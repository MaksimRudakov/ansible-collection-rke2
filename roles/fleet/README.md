# maksimrudakov.rke2.fleet

Installs [Rancher Fleet](https://fleet.rancher.io/) standalone (without Rancher Manager) on the cluster: adds the charts.rancher.io helm repo, installs the `fleet-crd` and `fleet` charts, waits for the `fleet-controller` rollout and optionally registers a `GitRepo` resource with a basic-auth secret — a minimal GitOps loop right after cluster deployment.

Helm and kubectl run on the target node itself, so the role is meant for a server node; defaults point at the RKE2 paths (`/etc/rancher/rke2/rke2.yaml`, RKE2-bundled kubectl). Any other cluster works by overriding `fleet_kubeconfig` / `fleet_kubectl_bin`. The helm binary is installed automatically (`fleet_helm_install: false` if it is already there).

The collection ships the `maksimrudakov.rke2.fleet` playbook that runs this role on the first server of the `rke2_servers` group.

## Variables

Full list with types and descriptions: [`meta/argument_specs.yml`](meta/argument_specs.yml). Key ones: `fleet_version`, `fleet_gitrepo_enabled` + `fleet_gitrepo_url` / `fleet_gitrepo_token` / `fleet_gitrepo_paths`, `fleet_extra_values`, `fleet_helm_url` (air-gapped mirror).

```yaml
- hosts: rke2_servers[0]
  become: true
  roles:
    - role: maksimrudakov.rke2.fleet
      vars:
        fleet_version: "0.11.1"
        fleet_gitrepo_enabled: true
        fleet_gitrepo_url: https://gitlab.example.com/infra/fleet-manifests.git
        fleet_gitrepo_token: "{{ vault_fleet_git_token }}"
        fleet_gitrepo_paths:
          - clusters/dev
```

An empty `fleet_gitrepo_token` skips the secret and applies the `GitRepo` without `clientSecretName` (public repository).

## Tags

| Tag | Purpose |
|-----|---------|
| `install` | Helm repo + fleet-crd/fleet charts + controller rollout wait |
| `gitrepo` | Only re-apply the GitRepo and auth secret (path changes, token rotation) |

```bash
ansible-playbook maksimrudakov.rke2.fleet -i inventory/<env>/ -t gitrepo
```

## License

Apache-2.0
