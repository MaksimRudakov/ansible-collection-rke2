# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

Ansible collection `maksimrudakov.rke2`: deploys and operates RKE2 Kubernetes clusters (online and air-gapped) with Cilium as the tuned-default CNI. One node-level role (`roles/node`) plus orchestration playbooks (`playbooks/`). The repo must live at the canonical collection path `<...>/ansible_collections/maksimrudakov/rke2` with `ANSIBLE_COLLECTIONS_PATH` pointing at the `collections` root, or FQCN resolution breaks.

## Commands

```bash
yamllint .
ansible-lint                # production profile; skip-list in .ansible-lint

molecule test -s airgap     # fast: mock mirror, URL rewrites, downgrade guard (no real RKE2)
molecule test               # full: real single-node RKE2 bootstrap in a privileged container
molecule test -s ha         # 1 server + 1 agent via the real deploy playbook; NOT in CI,
                            # unreliable on WSL2/slow disks (etcd fsync) — run on real hosts

# Iterating on a single scenario without full destroy/create cycle:
molecule converge -s airgap && molecule verify -s airgap

ansible-galaxy collection build   # tarball, same as CI build job
```

Deps: `pip install "molecule>=25.1" "molecule-plugins[docker]>=23.7" "ansible-core>=2.16" ansible-lint yamllint` and `ansible-galaxy collection install ansible.posix community.docker`.

CI (`.github/workflows/ci.yaml`): ansible-lint → molecule matrix (airgap, default) + collection build. Must be green before review.

## Architecture

**Separation of concerns:** orchestration (serial, delegation order, play sequencing) lives in `playbooks/`; node-level convergence lives in `roles/node`. Cluster topology is defined entirely by inventory — playbooks never change per environment.

**Inventory contract:** groups `rke2_servers` (control-plane) and `rke2_agents`. The FIRST host of `rke2_servers` is the bootstrap node (`rke2_bootstrap_host` / `rke2_first_server` in `defaults/main.yml`). New node classes are child groups of `rke2_agents` with their own `rke2_node_labels`/`rke2_node_taints`.

**Role flow** (`roles/node/tasks/main.yml`): validate → derive `rke2_server_url` from bootstrap host + slurp join token over delegation (`token.yml`) → `install.yml` → `config.yml` → one of `first_server.yml` / `additional_server.yml` / `agent.yml` depending on `rke2_role` + `rke2_first_server`. No `hostvars` plumbing in playbooks.

**Restart guard:** the `Restart rke2` handler only fires when the service was already active before the play (`rke2_service_active` fact from `service_state.yml`). This avoids a double restart on fresh bootstrap. `tasks_from: config` populates the guard itself when included standalone. Server plays must run `serial: 1` — parallel control-plane restarts break etcd quorum.

**Day-2 entry points:** the role exposes reusable task files for custom playbooks via `tasks_from`: `token`, `cordon`, `drain`, `uncordon`, `wait_ready`, `rotate_certs`, `config`. Delegation is variable-driven (`rke2_kubectl_delegate`, default — bootstrap host; the upgrade playbook overrides it to "another server" while a server restarts).

**Air-gapped mode:** `rke2_airgap: true` + `rke2_mirror_base` fetches the install script from the mirror and rewrites upstream URLs inside it (`rke2_airgap_url_rewrites`). Downgrades are refused unless `rke2_allow_downgrade: true`.

**CNI-agnostic:** `rke2_cni` defaults to `cilium`; `rke2_disable_kube_proxy` is derived (`true` only for cilium). Cilium tuning goes through `rke2_cilium_values` → HelmChartConfig template. Any other CNI/config is covered by `rke2_extra_config` (merged into config.yaml as-is) and `rke2_manifests` (arbitrary manifests dropped into the server manifests dir).

**Molecule quirks (do not remove):** the `default` scenario runs real RKE2 in docker and needs the `/var/lib/rancher` anonymous volume (overlayfs-on-overlayfs), privileged + `cgroupns_mode: host`, `/dev/kmsg`, and the prepare-step workarounds. Plays that are informational-only carry `tags: [molecule-notest]`.

## Conventions

- 2 spaces, YAML without redundant quotes, FQCN module names, idempotency: re-running any playbook on a converged cluster must be a no-op.
- All role variables use the `rke2_` prefix (not the role-name prefix — deliberate, see `.ansible-lint` skip). Every new variable goes into `roles/node/meta/argument_specs.yml` AND README.
- `no_log: "{{ rke2_no_log }}"` on any task touching the token or registry credentials.
- Conventional Commits (`feat:`, `fix:`, ...). Update `CHANGELOG.md` under `## [Unreleased]` in every PR.
- Release: bump `version` in `galaxy.yml`, move Unreleased → `## [X.Y.Z]` in CHANGELOG, push signed tag `vX.Y.Z`; the release workflow builds and publishes to Galaxy.
