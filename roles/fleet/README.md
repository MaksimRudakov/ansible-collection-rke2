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

## Rancher Manager on top

Fleet can be installed standalone **before** Rancher Manager — to roll out what Rancher itself needs (load balancer, cert-manager, ingress) — and handed over when Rancher is installed. Rancher then helm-upgrades `fleet`/`fleet-crd` with its own values and moves the local-cluster agent to `cattle-fleet-local-system`. Two things make the handover clean:

```yaml
fleet_version: "110.0.2+up0.16.2"            # = fleetVersion in build.yaml of your Rancher release
fleet_agent_namespace: cattle-fleet-local-system
```

- **`fleet_agent_namespace`** — the agent namespace is also its `AGENT_SCOPE`, and the scope is part of the ownership id (`objectset.rio.cattle.io/id`) of every object Fleet deploys. With the chart default (empty scope) every object deployed before Rancher becomes "not owned by us" after the handover: bundles go `Modified`, nothing is applied any more. Set it from the start; changing it on a cluster with deployed bundles causes exactly that.
- **`fleet_version`** equal to the Fleet bundled with Rancher — the takeover then is a same-version upgrade.

After the handover the role leaves the charts alone (`fleet_rancher_managed: auto` finds `cattle-system/rancher`) and only applies the GitRepo — upgrade Fleet together with Rancher.

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
