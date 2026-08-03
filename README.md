# Ansible Collection: maksimrudakov.rke2

[![CI](https://github.com/MaksimRudakov/ansible-collection-rke2/actions/workflows/ci.yaml/badge.svg)](https://github.com/MaksimRudakov/ansible-collection-rke2/actions/workflows/ci.yaml)
[![Release](https://img.shields.io/github/v/release/MaksimRudakov/ansible-collection-rke2?sort=semver)](https://github.com/MaksimRudakov/ansible-collection-rke2/releases)
[![License: Apache-2.0](https://img.shields.io/badge/License-Apache_2.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)
[![Ansible Galaxy](https://img.shields.io/ansible/collection/v/maksimrudakov/rke2?label=galaxy)](https://galaxy.ansible.com/ui/repo/published/maksimrudakov/rke2/)

Deploys and operates [RKE2](https://docs.rke2.io/) Kubernetes clusters with [Cilium](https://cilium.io/) CNI. Works both **online** and **fully air-gapped** (through an internal mirror such as a Nexus raw proxy).

The collection ships a node-level role plus ready-to-run playbooks — the whole cluster topology is defined by inventory, the playbooks never change.

## Contents

| Component | FQCN | Purpose |
|-----------|------|---------|
| role | `maksimrudakov.rke2.node` | Converge one node: install, config, join. Entry points for day-2 helpers (`token`, `cordon`, `drain`, `uncordon`, `stop`, `start`, `node_roles`, `wait_ready`, `rotate_certs`, `config`) |
| playbook | `maksimrudakov.rke2.deploy` | Full cluster bootstrap: first server → additional servers (`serial: 1`) → agents |
| playbook | `maksimrudakov.rke2.upgrade` | Rolling upgrade: cordon → drain → converge → wait Ready → uncordon |
| playbook | `maksimrudakov.rke2.reconfig` | Rolling config reapply (kubelet args, labels, registries) without drain |
| playbook | `maksimrudakov.rke2.rotate_certs` | Rotate kube-apiserver serving certs after `rke2_tls_san` change |
| playbook | `maksimrudakov.rke2.remove_node` | Remove node(s): cordon → drain → delete Node → `rke2-uninstall.sh`. Explicit target via `-e rke2_remove_hosts=<pattern>` |
| playbook | `maksimrudakov.rke2.clustermesh_connect` | Connect two RKE2 clusters into a Cilium ClusterMesh (run with both inventories: `-i a/ -i b/`). Needs the cilium CLI on the controller |

## Requirements

- ansible-core >= 2.15, `ansible.posix` collection (installed as dependency)
- Ubuntu 22.04/24.04 or EL 8/9 targets with systemd

## Installation

```bash
ansible-galaxy collection install maksimrudakov.rke2
```

Or straight from git:

```bash
ansible-galaxy collection install git+https://github.com/MaksimRudakov/ansible-collection-rke2.git,v1.0.0
```

## Inventory contract

Two canonical groups; node types are pure inventory — a new node class is a child group of `rke2_agents` with its own labels/taints:

```yaml
rke2_cluster:
  children:
    rke2_servers:        # control-plane; the FIRST host bootstraps the cluster
      hosts:
        master-01: { ansible_host: <MASTER_01_IP> }
        master-02: { ansible_host: <MASTER_02_IP> }
        master-03: { ansible_host: <MASTER_03_IP> }
    rke2_agents:
      children:
        workers:
          hosts:
            worker-01: { ansible_host: <WORKER_01_IP> }
            worker-02: { ansible_host: <WORKER_02_IP> }
            worker-03: { ansible_host: <WORKER_03_IP> }
        runners:         # dedicated GitLab runner node
          hosts:
            runner-01: { ansible_host: <RUNNER_01_IP> }
          vars:
            rke2_node_labels: ["node-type=gitlab-runner"]
            rke2_node_taints: ["dedicated=gitlab-runner:NoSchedule"]
```

Full example: [`examples/inventory/`](examples/inventory/).

Joining nodes derive the registration URL from the first server and fetch the join token over delegation automatically — no `hostvars` plumbing in playbooks. For production set a stable `rke2_server_url` (DNS RR / VIP) in group_vars; the bootstrap server ignores it during initial bootstrap.

## Usage

```bash
# Deploy the whole cluster (idempotent, re-run converges config)
ansible-playbook maksimrudakov.rke2.deploy -i inventory/prod/

# Rolling upgrade
ansible-playbook maksimrudakov.rke2.upgrade -i inventory/prod/ -e rke2_version=<VERSION>

# Rolling config reapply (kubelet args, labels, registries)
ansible-playbook maksimrudakov.rke2.reconfig -i inventory/prod/

# Certificate rotation after changing rke2_tls_san
ansible-playbook maksimrudakov.rke2.rotate_certs -i inventory/prod/
```

Role variables: see [`roles/node/meta/argument_specs.yml`](roles/node/meta/argument_specs.yml). Key ones: `rke2_version`, `rke2_airgap` + `rke2_mirror_base`, `rke2_registry_mirrors` / `rke2_registry_configs`, `rke2_cilium_values`, `rke2_extra_config`, `rke2_manifests`, `rke2_node_roles`, `rke2_etcd_snapshot_schedule_cron` / `rke2_etcd_snapshot_retention`, `rke2_cis_profile`, `rke2_allow_downgrade`.

### Beyond Cilium

Cilium is the tuned default, not a requirement. Any CNI works: `rke2_cni: canal` (or `calico`, `none`, `multus,cilium`) — `rke2_disable_kube_proxy` automatically stays `false` for non-Cilium CNIs. Anything RKE2's `config.yaml` supports goes into `rke2_extra_config` (rendered as-is), and `rke2_manifests` drops arbitrary manifests into the server manifests directory — `HelmChartConfig` for any packaged component, your own `HelmChart`, plain resources:

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

### Cilium ClusterMesh

Two clusters deployed from separate inventories (distinct `cluster.name`/`cluster.id` and non-overlapping CIDRs in `rke2_cilium_values`, clustermesh apiserver enabled) are connected with a single run:

```bash
ansible-playbook maksimrudakov.rke2.clustermesh_connect -i inventory/dc1/ -i inventory/dc2/
```

Cluster names are read from the nodes themselves (with two merged inventories the group_vars of same-named groups override each other), kubeconfigs land in `$PWD/kubeconfig/` (override with `rke2_mesh_kubeconfig_dir`; keep out of VCS). RKE2 specifics are handled: the `rke2-cilium` helm release name and `--allow-mismatching-ca` for the default per-cluster CAs (`rke2_mesh_allow_mismatching_ca: false` once you ship a shared CA). Note: on RKE2 the cilium cluster-pool IPAM does not read `cluster-cidr` — set `ipam.operator.clusterPoolIPv4PodCIDRList` in `rke2_cilium_values` explicitly per cluster.

## Day-2 helpers (role entry points)

For custom playbooks, the role exposes reusable task files:

```yaml
- name: Drain node before maintenance
  ansible.builtin.include_role:
    name: maksimrudakov.rke2.node
    tasks_from: drain   # also: token, cordon, uncordon, stop, start, node_roles, delete_node, uninstall, wait_ready, rotate_certs, config
```

Delegation and paths are variables: `rke2_kubectl_delegate` (which host runs kubectl, default — first server), `rke2_node_name`, `rke2_data_dir`, `rke2_kubectl`, `rke2_kubeconfig`, `rke2_drain_timeout`, `rke2_ready_timeout`.

## Migrating from the standalone `maksimrudakov.rke2` role

- Role FQCN is now `maksimrudakov.rke2.node`.
- The restart handler is named `Restart rke2` and only fires when the service was already running; `tasks_from: config` now populates that guard itself.
- `rke2_first_server` defaults to "first host of `rke2_servers`" — inventories using other group names must keep setting it explicitly (previous behavior).
- `registries.yaml` is removed when no mirrors/configs are defined.

## Testing

```bash
export ANSIBLE_COLLECTIONS_PATH=<path-to>/collections
molecule test -s airgap    # fast: mock mirror, URL rewrites, downgrade guard
molecule test              # full: single-node RKE2 bootstrap in a privileged container
```

Beyond molecule, every playbook and feature is verified end-to-end against real
HA clusters (RKE2 v1.34/v1.35, Ubuntu 24.04, Cilium and Canal, CIS mode, rolling
patch and minor upgrades, node add/remove) — the full matrix with versions is in
[`TESTING.md`](TESTING.md).

## License

Apache-2.0

## Author

Maxim Rudakov
