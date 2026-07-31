# maksimrudakov.rke2.node

Converges a single RKE2 node: installs the requested version (online or through an internal air-gapped mirror), renders `config.yaml` / `registries.yaml` / Cilium `HelmChartConfig`, starts the service and joins the node to the cluster as a server or agent.

Not meant to be used alone for whole-cluster operations — the collection ships playbooks (`maksimrudakov.rke2.deploy`, `upgrade`, `reconfig`, `rotate_certs`) that orchestrate this role with correct ordering, `serial: 1` for control-plane nodes and cordon/drain around restarts. See the [collection README](../../README.md).

## Variables

Full list with types and descriptions: [`meta/argument_specs.yml`](meta/argument_specs.yml). Required: `rke2_role` (`server` / `agent`). Joining nodes need `rke2_server_url` + `rke2_token`, or an inventory following the `rke2_servers` group contract for auto-derivation.

## Entry points

Besides `main`, the role exposes task files for custom day-2 playbooks via `include_role: tasks_from=...`:

| Entry point | Purpose |
|-------------|---------|
| `config` | Re-render config files, restart via handler if changed |
| `token` | Fetch the join token from the bootstrap host |
| `cordon` / `drain` / `uncordon` | Node lifecycle around maintenance |
| `stop` / `start` | Stop/start the rke2 service; `rke2_stop_killall: true` also runs `rke2-killall.sh` (a plain service stop leaves pods running by RKE2 design) |
| `node_roles` | Apply `node-role.kubernetes.io/<role>` labels from `rke2_node_roles` via kubectl (kubelet may not self-assign them) |
| `delete_node` | Delete the Node object via kubectl (etcd member removed automatically for servers) |
| `uninstall` | Wipe RKE2 from the host (`rke2-uninstall.sh` + leftover dirs); drain/delete the Node first |
| `wait_ready` | Wait for the Node object to become Ready |
| `rotate_certs` | Remove kube-apiserver serving certs for regeneration |

## License

Apache-2.0
