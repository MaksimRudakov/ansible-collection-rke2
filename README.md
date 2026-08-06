# Ansible Collection: maksimrudakov.rke2

[![CI](https://github.com/MaksimRudakov/ansible-collection-rke2/actions/workflows/ci.yaml/badge.svg)](https://github.com/MaksimRudakov/ansible-collection-rke2/actions/workflows/ci.yaml)
[![Release](https://img.shields.io/github/v/release/MaksimRudakov/ansible-collection-rke2?sort=semver)](https://github.com/MaksimRudakov/ansible-collection-rke2/releases)
[![License: Apache-2.0](https://img.shields.io/badge/License-Apache_2.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)
[![Ansible Galaxy](https://img.shields.io/ansible/collection/v/maksimrudakov/rke2?label=galaxy)](https://galaxy.ansible.com/ui/repo/published/maksimrudakov/rke2/)

Deploys and operates [RKE2](https://docs.rke2.io/) Kubernetes clusters: bootstrap, rolling upgrades, day-2 operations, multi-cluster [Cilium ClusterMesh](https://docs.cilium.io/en/stable/network/clustermesh/) and standalone [Rancher Fleet](https://fleet.rancher.io/) GitOps. **CNI-agnostic** — any CNI supported by RKE2 (canal, calico, multus, none, ...) works out of the box; Cilium ships as a tuned default. Installs **online** or **fully air-gapped** (through an internal mirror such as a Nexus raw proxy).

The collection ships node-level roles plus ready-to-run playbooks — the whole cluster topology is defined by inventory, the playbooks never change per environment.

- [Contents](#contents)
- [Requirements](#requirements)
- [Quickstart](#quickstart)
- [Inventory contract](#inventory-contract)
- [Dynamic inventories](#dynamic-inventories)
- [Configuration](#configuration)
- [CNI](#cni)
- [Cilium ClusterMesh](#cilium-clustermesh)
- [Rancher Fleet (GitOps)](#rancher-fleet-gitops)
- [Day-2 helpers](#day-2-helpers-role-entry-points)
- [Versioning](#versioning)
- [Testing](#testing)

## Contents

| Component | FQCN | Purpose |
|-----------|------|---------|
| role | `maksimrudakov.rke2.node` | Converge one node: install, config, join. Reusable entry points for day-2 operations |
| role | `maksimrudakov.rke2.fleet` | Rancher Fleet standalone GitOps: fleet-crd/fleet charts + optional `GitRepo` with auth secret |
| playbook | `maksimrudakov.rke2.deploy` | Full cluster bootstrap: first server → additional servers (`serial: 1`) → agents |
| playbook | `maksimrudakov.rke2.upgrade` | Rolling upgrade: cordon → drain → converge → wait Ready → uncordon |
| playbook | `maksimrudakov.rke2.reconfig` | Rolling config reapply (kubelet args, labels, registries) without drain |
| playbook | `maksimrudakov.rke2.rotate_certs` | Rotate kube-apiserver serving certs after `rke2_tls_san` change |
| playbook | `maksimrudakov.rke2.remove_node` | Remove node(s): graceful (cordon → drain → delete → uninstall) or dead-host cleanup |
| playbook | `maksimrudakov.rke2.clustermesh_connect` | Connect two RKE2 clusters into a Cilium ClusterMesh |
| playbook | `maksimrudakov.rke2.fleet` | Install Fleet on the first server; `-t gitrepo` re-applies only the GitRepo |

## Requirements

- ansible-core >= 2.15 (tested with 2.16), `ansible.posix` collection (installed as a dependency)
- Ubuntu 22.04/24.04 or EL 8/9 targets with systemd

## Quickstart

```bash
ansible-galaxy collection install maksimrudakov.rke2
```

Minimal inventory (`inventory/hosts.yml`):

```yaml
rke2_cluster:
  children:
    rke2_servers:
      hosts:
        master-01: {ansible_host: <MASTER_IP>}
    rke2_agents:
      hosts:
        worker-01: {ansible_host: <WORKER_IP>}
```

```bash
ansible-playbook maksimrudakov.rke2.deploy -i inventory/
```

That is a working cluster. Everything below is about shaping it: HA, node classes, CNI, registries, upgrades.

## Inventory contract

Two canonical groups: `rke2_servers` (control-plane) and `rke2_agents`. The **first host** of `rke2_servers` bootstraps the cluster; joining nodes derive the registration URL from it and fetch the join token over delegation automatically — no `hostvars` plumbing. A new node class (GPU nodes, ingress nodes, CI runners) is just a child group of `rke2_agents` with its own `rke2_node_labels` / `rke2_node_taints` / `rke2_node_roles`.

Fully annotated examples distilled from a real multi-cluster stand:

- [`examples/inventory/`](examples/inventory/) — static inventory: HA control plane, node classes, production-grade group_vars with commentary on every decision
- [`examples/inventory-kubevirt/`](examples/inventory-kubevirt/) — dynamic inventory (KubeVirt/Harvester): VM labels → collection groups

For production set a stable `rke2_server_url` (DNS RR / VIP) in group_vars; the bootstrap server ignores it during initial bootstrap.

## Dynamic inventories

The contract maps cleanly onto dynamic inventory plugins (KubeVirt, AWS, OpenStack, ...): build `rke2_servers`/`rke2_agents` from instance labels with the plugin's `groups:` conditionals, and keep the `rke2_cluster` hierarchy in a small static file next to the plugin config. Hard-earned rules:

- **Bootstrap ordering**: `rke2_servers[0]` is the bootstrap node and plugins add hosts in discovery order (usually alphabetical). Name your servers so the intended bootstrap sorts first, or pin `rke2_first_server: true` on it explicitly.
- **Scope by labels, not by environment**: select instances with a dedicated label (e.g. `service_type=<cluster>`), not a broad environment selector — otherwise foreign instances with a matching `role` label can leak into your control-plane group.
- **One cluster = one inventory directory.** Never merge two clusters into one inventory for `deploy` — group_vars of same-named groups override each other (the `clustermesh_connect` playbook is the deliberate exception and reads cluster identity from the nodes instead).
- **Broken-inventory escape hatch**: day-2 playbooks accept an inline inventory (`-i '<SERVER_IP>,'`) when the dynamic source itself is down — see the `remove_node` dead-host path.

Working example: [`examples/inventory-kubevirt/`](examples/inventory-kubevirt/).

## Configuration

Canonical variable reference: [`roles/node/meta/argument_specs.yml`](roles/node/meta/argument_specs.yml) (and [`roles/fleet/meta/argument_specs.yml`](roles/fleet/meta/argument_specs.yml) for Fleet). By area:

| Area | Variables |
|------|-----------|
| Installation & air-gap | `rke2_version`, `rke2_airgap` + `rke2_mirror_base` + `rke2_airgap_url_rewrites`, `rke2_install_method`, `rke2_binary_search_paths`, `rke2_allow_downgrade` |
| Topology & API access | `rke2_server_url`, `rke2_token`, `rke2_tls_san`, `rke2_servers_group` / `rke2_first_server` |
| Networking & CNI | `rke2_cni`, `rke2_disable_kube_proxy` (derived), `rke2_cluster_cidr`, `rke2_service_cidr`, `rke2_cilium_values`, `rke2_extra_config`, `rke2_manifests` |
| Node classes | `rke2_node_labels`, `rke2_node_taints`, `rke2_node_roles` (ROLES column via kubectl — kubelet may not self-assign those), `rke2_kubelet_arg`, per-component args |
| Registries | `rke2_registry_mirrors`, `rke2_registry_configs` |
| etcd | `rke2_etcd_snapshot_schedule_cron` / `_retention` / `_dir`, `rke2_etcd_expose_metrics` |
| Security | `rke2_cis_profile` (the role provisions the etcd user and CIS sysctl automatically), `rke2_write_kubeconfig_mode`, `rke2_no_log` |
| Day-2 plumbing | `rke2_kubectl_delegate`, `rke2_node_name`, `rke2_drain_timeout`, `rke2_ready_timeout` |

Every variable is demonstrated with commentary in [`examples/inventory/group_vars/rke2_cluster.yml`](examples/inventory/group_vars/rke2_cluster.yml).

## CNI

`rke2_cni` accepts anything RKE2 supports: `cilium` (default), `canal`, `calico`, `none`, `multus,cilium`, ... `rke2_disable_kube_proxy` is derived automatically — `true` only for Cilium (kube-proxy replacement), any other CNI keeps kube-proxy instead of silently losing service routing.

Component tuning is uniform for every CNI: `rke2_extra_config` merges arbitrary keys into `config.yaml`, and `rke2_manifests` drops manifests into the server manifests directory — `HelmChartConfig` for any packaged component, your own `HelmChart`, plain resources:

```yaml
rke2_cni: canal
rke2_manifests:
  - name: rke2-canal-config
    content:
      apiVersion: helm.cattle.io/v1
      kind: HelmChartConfig
      metadata: {name: rke2-canal, namespace: kube-system}
      spec:
        valuesContent: |-
          flannel:
            backend: vxlan
```

Cilium is tuned through `rke2_cilium_values` → HelmChartConfig. Note that setting it **replaces** the role defaults entirely (override, not merge) — start from the default block in the [example group_vars](examples/inventory/group_vars/rke2_cluster.yml).

## Cilium ClusterMesh

Two clusters deployed from separate inventories (distinct `cluster.name`/`cluster.id` and non-overlapping CIDRs in `rke2_cilium_values`, clustermesh apiserver enabled) are connected with a single run:

```bash
ansible-playbook maksimrudakov.rke2.clustermesh_connect -i inventory/dc1/ -i inventory/dc2/
```

Cluster names are read from the nodes themselves (with two merged inventories the group_vars of same-named groups override each other), kubeconfigs land in `$PWD/kubeconfig/` (override with `rke2_mesh_kubeconfig_dir`; keep out of VCS). RKE2 specifics are handled: the `rke2-cilium` helm release name and `--allow-mismatching-ca` for the default per-cluster CAs (`rke2_mesh_allow_mismatching_ca: false` once you ship a shared CA). Note: on RKE2 the cilium cluster-pool IPAM does not read `cluster-cidr` — set `ipam.operator.clusterPoolIPv4PodCIDRList` in `rke2_cilium_values` explicitly per cluster.

## Rancher Fleet (GitOps)

`maksimrudakov.rke2.fleet` installs [Fleet](https://fleet.rancher.io/) standalone (no Rancher Manager) on the first server and optionally registers a `GitRepo` — a minimal GitOps loop right after `deploy`:

```yaml
fleet_version: "110.0.0+up0.16.0"   # chart version as on charts.rancher.io
fleet_gitrepo_enabled: true
fleet_gitrepo_url: https://gitlab.example.com/infra/fleet-manifests.git
fleet_gitrepo_token: "{{ vault_fleet_git_token }}"
fleet_gitrepo_paths: [clusters/prod]
```

Empty `fleet_gitrepo_token` — public repo, no secret. `-t gitrepo` re-applies only the GitRepo (paths, token rotation). Re-runs with a pinned `fleet_version` are idempotent; a version bump in group_vars is the upgrade path. When uninstalling, delete GitRepos and let bundles clean up **before** removing the controller (see [TESTING.md](TESTING.md)). Variables: [`roles/fleet/meta/argument_specs.yml`](roles/fleet/meta/argument_specs.yml).

## Day-2 helpers (role entry points)

For custom playbooks, `maksimrudakov.rke2.node` exposes reusable task files — `token`, `cordon`, `drain`, `uncordon`, `stop`, `start`, `node_roles`, `delete_node`, `uninstall`, `wait_ready`, `rotate_certs`, `config`:

```yaml
- name: Drain node before maintenance
  ansible.builtin.include_role:
    name: maksimrudakov.rke2.node
    tasks_from: drain
```

Full table with per-entry-point notes: [`roles/node/README.md`](roles/node/README.md). Delegation and paths are variable-driven (`rke2_kubectl_delegate` — which host runs kubectl, default the bootstrap host; the upgrade playbook overrides it to "another server" while a server restarts).

## Versioning

[SemVer](https://semver.org/) + [Keep a Changelog](https://keepachangelog.com/): breaking changes only in major releases, every change lands in [CHANGELOG.md](CHANGELOG.md). Upgrading the collection never touches your clusters by itself — changes apply on the next playbook run.

## Testing

```bash
export ANSIBLE_COLLECTIONS_PATH=<path-to>/collections
molecule test -s airgap    # fast: mock mirror, URL rewrites, downgrade guard
molecule test              # full: single-node RKE2 + Fleet in a privileged container
```

Beyond molecule, every playbook and feature is verified end-to-end against real
HA clusters (RKE2 v1.34/v1.35, Ubuntu 24.04, Cilium and Canal, CIS mode, rolling
patch and minor upgrades, node add/remove, ClusterMesh, Fleet chart upgrades) —
the full matrix with versions is in [`TESTING.md`](TESTING.md).

## License

Apache-2.0

## Author

Maxim Rudakov
