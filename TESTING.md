# Testing

## Automated (CI)

Every PR runs ansible-lint (production profile), yamllint and two molecule scenarios ([`.github/workflows/ci.yaml`](.github/workflows/ci.yaml)):

| Scenario | What it covers |
|----------|----------------|
| `airgap` | Mock mirror: install.sh URL rewrites, installer env, script cleanup, downgrade guard. No real RKE2 |
| `default` | Real single-node RKE2 bootstrap in a privileged Ubuntu 24.04 container: converge, idempotence, node Ready, Cilium up |
| `ha` (opt-in, not in CI) | 1 server + 1 agent via the real `deploy` playbook: bootstrap, token/URL auto-derivation, agent join, labels/taints. Needs fast disks — see scenario header |

## Verified end-to-end on real clusters (v1.2.0 cycle, 2026-07)

Everything below was exercised against a live stand: **8 KubeVirt/Harvester VMs** (3 servers + 4 workers + 1 dedicated runner node, 2 vCPU / 8 GB / Ubuntu 24.04.4 LTS, kernel 6.8), **ansible-core 2.16.14**, dynamic `kubevirt.core.kubevirt` inventory mapped to the `rke2_servers`/`rke2_agents` contract.

### Cluster lifecycle

| Scenario | Versions / result |
|----------|-------------------|
| `deploy` from scratch, HA (Cilium) | RKE2 `v1.34.3+rke2r3`; 3-server etcd quorum, serial joins, automatic token/URL derivation |
| `deploy` from scratch, HA (Canal) | RKE2 `v1.35.6+rke2r1`; `rke2_disable_kube_proxy` derived to `false` (kube-proxy present on every node), flannel `vxlan` backend applied via `rke2_manifests` HelmChartConfig |
| `deploy` single node → grow to HA | 1 server bootstrap, then incremental growth to 8 nodes with the same playbook |
| Idempotence | Re-run of `deploy` on every converged topology: `changed=0` on all nodes |
| `upgrade`, patch | `v1.34.3+rke2r3` → `v1.34.9+rke2r1`, rolling (cordon → drain → converge → Ready → uncordon), zero failures |
| `upgrade`, minor (k8s 1.34 → 1.35) | `v1.34.9+rke2r1` → `v1.35.6+rke2r1`, same rolling flow |
| `reconfig` | kubelet args (image GC thresholds, eviction hard/soft) verified via kubelet `configz`; registry mirrors + auth (Nexus docker proxy) verified via containerd `hosts.toml` and live pulls |
| `rotate_certs` | New `rke2_tls_san` entry present in the apiserver serving cert of every server afterwards |
| `remove_node`, graceful | cordon → drain → Node deleted → host wiped by `rke2-uninstall.sh`; workloads rescheduled |
| `remove_node`, dead host | Node object of an already-wiped host removed from a live server; also exercised with a **broken dynamic inventory** via inline `-i '<SERVER_IP>,'` |
| Scale-out | New labeled VM joined by re-running `deploy` — zero config changes; node previously wiped by `uninstall` re-joined the same way |
| Mass teardown | `tasks_from: uninstall` across all 8 nodes at once |

### Features

| Feature | Result |
|---------|--------|
| `rke2_node_roles` | ROLES column reconciliation: adds missing `node-role.kubernetes.io/*` labels, removes stale ones, never touches RKE2-managed `control-plane`/`etcd`/`master`; idempotent |
| `rke2_cis_profile: cis` | Without the role's CIS prerequisites rke2-server fails (`missing required: user: unknown user etcd`) — the role now provisions the etcd user and `rke2-cis-sysctl.conf`; verified single-node and HA; PodSecurity `restricted:latest` enforced (privileged pod rejected), restricted-compliant workload runs clean |
| etcd snapshots | `schedule_cron`/`retention` land in server config; manual `rke2 etcd-snapshot save` + `list` verified |
| `stop` / `start` entry points | Service stop → node NotReady → start → Ready again |
| Cilium (default values) | kube-proxy replacement, Hubble relay/UI up, Prometheus metrics live on `:9962` (cilium_*) and `:9965` (hubble_* with workload context labels) |
| Registry mirrors | docker.io / quay.io / registry.k8s.io / gcr.io / ghcr.io through a Nexus group proxy with auth; pull-through verified |

### Cilium ClusterMesh (unreleased, 2026-08)

Two clusters from separate inventories (dc1 3+2 HA / dc2 1+1, different L3 subnets, non-overlapping CIDRs 10.10/10.20), connected with `maksimrudakov.rke2.clustermesh_connect`: `cilium connectivity test --multi-cluster` — 81/82 passed (the one failure is the log-scan catching transient connect-window warnings), cross-cluster global services verified, mesh survived scaling dc1 from 1+1 to 3+2 without reconnection.

### Rancher Fleet role (unreleased, 2026-08)

- Molecule: the `default` scenario installs Fleet on the real single-node RKE2 cluster (pinned chart), the idempotence phase re-runs the role (must be `changed=0` — exercises the version guard), verify asserts the controller rollout and the `fleet-agent-local` bundle readiness.
- Stand e2e: fresh install (both clusters), private GitRepo with PAT, helm bundle deploys, GitOps update loop, per-cluster paths, `-t gitrepo` re-apply, **chart upgrade `109.0.5+up0.15.5` → `110.0.0+up0.16.0`** — GitRepo and bundles survive, deployed workloads keep running untouched (same pod).
- Uninstall order caveat (documented): delete GitRepos and let bundles clean up BEFORE uninstalling the controller — otherwise `bundledeployment` finalizers hang namespaces with nobody to process them.

### Not yet covered end-to-end

- EL 8/9 targets (Ubuntu only so far; the rpm install path is exercised in neither molecule nor the stand)
- Real air-gapped install (`rke2_airgap: true` is covered by the molecule mock only)
- etcd snapshot **restore** (disaster recovery is intentionally out of playbook scope)
- ansible-core versions other than 2.16
