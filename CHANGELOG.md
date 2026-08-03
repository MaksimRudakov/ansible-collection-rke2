# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/) and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

## [1.4.0] - 2026-08-03

### Added
- `maksimrudakov.rke2.fleet` role and playbook: Rancher Fleet standalone (no Rancher Manager) — installs the `fleet-crd`/`fleet` charts from charts.rancher.io via node-local helm/kubectl, waits for the fleet-controller rollout and optionally registers a `GitRepo` with a basic-auth secret (empty token — public repo, no secret). Helm binary is installed automatically with checksum verification and amd64/arm64 detection (`fleet_helm_install: false` to skip, `fleet_helm_url` for air-gapped mirrors). Tags `install` / `gitrepo` allow token rotation and path updates without touching the installation. Re-runs with a pinned `fleet_version` are idempotent (the deployed chart is compared and the upgrade is skipped; `fleet_force_upgrade` pushes values changes); manifests with credentials go through mkstemp'ed files, never predictable `/tmp` paths. Covered by the `default` molecule scenario (install + idempotence guard + controller/agent verify); chart upgrades verified end-to-end (see TESTING.md).

## [1.3.0] - 2026-08-03

### Added
- `maksimrudakov.rke2.clustermesh_connect` playbook: connect two RKE2 clusters into a Cilium ClusterMesh — collects kubeconfigs from both server nodes (cluster names derived from the rendered HelmChartConfig, immune to two-inventory group_vars merging), merges contexts and drives `cilium clustermesh connect` / `status --wait` with the RKE2 specifics baked in (`rke2-cilium` helm release name, `--allow-mismatching-ca` by default). Requires the cilium CLI on the controller.

## [1.2.0] - 2026-07-31

### Fixed
- `rke2_cis_profile` was unusable: RKE2 in CIS mode requires the `etcd` system user and the shipped CIS sysctl profile, so `rke2-server` failed to start with `missing required: user: unknown user etcd`. When the profile is set, the role now creates the etcd user/group on servers and applies `rke2-cis-sysctl.conf` (tar and rpm layouts) before the service starts.
- Standalone `tasks_from: config` (the `reconfig` playbook path) rendered agent configs with an empty `server:` / `token:` — the join-parameter derivation lived only in `main.yml`, so the rke2-agent service failed to restart with "--server is required" (additional servers silently lost the `server:`/`token:` lines instead). Derivation is extracted to `join_params.yml` and included from both `main.yml` and `config.yml`.

### Added
- `maksimrudakov.rke2.remove_node` playbook with two explicit paths: graceful (`-e rke2_remove_hosts=<pattern>`: cordon → drain → delete the Node object → wipe the host with `rke2-uninstall.sh`; refuses to remove the kubectl delegate) and dead-host (`-e rke2_remove_dead_nodes=<name,...>`: delete Node objects from a live server when the VM is already gone — works even with a broken dynamic inventory via an inline `-i '<SERVER_IP>,'`). Building blocks available as role entry points `delete_node` and `uninstall`.
- etcd snapshot settings (servers only): `rke2_etcd_snapshot_schedule_cron`, `rke2_etcd_snapshot_retention`, `rke2_etcd_snapshot_dir`. Empty values keep RKE2 defaults; S3 upload via `rke2_extra_config`.
- `rke2_node_roles`: node roles for the ROLES column (`node-role.kubernetes.io/<role>` labels), e.g. `["worker"]`. Kubelet may not self-assign these (NodeRestriction), so the role reconciles them with kubectl over delegation after the node joins — missing roles are added, roles absent from the list are removed (RKE2-managed `control-plane`/`etcd`/`master` are never touched); also available standalone as the `node_roles` entry point.
- Role entry points `stop` / `start`: stop or start the `rke2-server` / `rke2-agent` service on a node. `rke2_stop_killall: true` additionally runs `rke2-killall.sh` after the stop — a plain service stop leaves pods and containers running by RKE2 design.

## [1.1.1] - 2026-07-30

### Fixed
- Examples and the `ha` molecule scenario used `node-role.kubernetes.io/runner=true` in `rke2_node_labels` — kubelet refuses to self-assign labels in that namespace (NodeRestriction) and crash-loops. Replaced with a plain label; the restriction is now documented for `rke2_node_labels`.

## [1.1.0] - 2026-07-30

### Added
- `rke2_manifests`: deploy arbitrary manifests to the server manifests directory (HelmChartConfig for any packaged component, HelmChart, plain resources) — the role is no longer Cilium-only for component tuning.
- Molecule `ha` scenario: 1 server + 1 agent deployed by the real `deploy` playbook — covers bootstrap, automatic token/URL derivation, agent join and runner labels/taints. 3-server etcd does not survive docker fsync latency; test additional-server joins on real hosts.

### Changed
- `rke2_disable_kube_proxy` default is now derived: `true` only when `rke2_cni` is `cilium`. Non-Cilium CNIs keep kube-proxy instead of silently losing service routing.

## [1.0.0] - 2026-07-30

Initial collection release. Supersedes the standalone `maksimrudakov.rke2` role (v1.1.0), which is included as `maksimrudakov.rke2.node`.

### Added
- Playbooks: `deploy`, `upgrade` (rolling, cordon/drain/wait/uncordon), `reconfig` (rolling, no drain), `rotate_certs`.
- Inventory contract `rke2_servers` / `rke2_agents`: the first server auto-bootstraps, joining nodes derive `rke2_server_url` and fetch the token from the bootstrap host automatically.
- Role entry points for day-2 operations: `token`, `cordon`, `drain`, `uncordon`, `wait_ready`, `rotate_certs`.
- Path variables `rke2_data_dir` / `rke2_kubectl` / `rke2_kubeconfig` instead of hardcoded paths.
- Example inventory: 3 servers + 3 workers + dedicated GitLab runner node.
- CI: ansible-lint, molecule matrix (`airgap` + `default`), collection build; release workflow with tag/version check and optional Galaxy publish.

### Changed
- `tasks_from: config` populates the restart-guard fact itself, so config-only runs restart a running service correctly.
- `config-server.yaml.j2` omits `server:` on the bootstrap node — `rke2_server_url` is safe to set cluster-wide.

[Unreleased]: https://github.com/MaksimRudakov/ansible-collection-rke2/compare/v1.1.1...HEAD
[1.1.1]: https://github.com/MaksimRudakov/ansible-collection-rke2/compare/v1.1.0...v1.1.1
[1.1.0]: https://github.com/MaksimRudakov/ansible-collection-rke2/compare/v1.0.0...v1.1.0
[1.0.0]: https://github.com/MaksimRudakov/ansible-collection-rke2/releases/tag/v1.0.0
