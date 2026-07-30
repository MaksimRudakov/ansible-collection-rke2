# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/) and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

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
