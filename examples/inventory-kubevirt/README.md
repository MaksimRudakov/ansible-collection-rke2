# Dynamic inventory example: KubeVirt / Harvester

VM labels drive the whole topology — adding a node to the cluster is
"create a VM with the right labels, re-run deploy". The same pattern maps
onto any dynamic inventory plugin with `compose`/`groups` support
(amazon.aws.aws_ec2, openstack.cloud.openstack, ...).

Label the VMs (in the VM template so VMIs inherit them):

| Label | Purpose |
|-------|---------|
| `tag.harvesterhci.io/service_type=<cluster-name>` | scopes THIS cluster — selection key |
| `tag.harvesterhci.io/role=master\|worker` | maps to rke2_servers / workers |

Files:

- `cluster.kubevirt.yml` — the plugin config: label selection + group mapping
- `static-groups.yml` — the rke2_cluster hierarchy (group_vars attachment point)
- `group_vars/rke2_cluster.yml` — cluster config (see ../inventory/ for the
  fully annotated version)

Hard-earned notes (each of these bit us on a real stand):

1. **Bootstrap ordering**: plugins add hosts in discovery order (usually
   alphabetical) and `rke2_servers[0]` bootstraps the cluster. Name servers
   so the intended bootstrap sorts first (master-01 < master-02), or pin
   `rke2_first_server: true` via host label/vars.
2. **Scope by a cluster-unique label**, not by a broad environment selector:
   with `environment=staging` any foreign VM carrying `role=master` lands in
   YOUR control-plane group.
3. **One cluster = one inventory directory.** Two clusters merged into one
   inventory make group_vars of the same-named groups override each other.
4. If the dynamic source (Harvester/cloud API) is down, day-2 playbooks
   still work with an inline inventory: `-i '<SERVER_IP>,'` (enable the
   host_list plugin if your ansible.cfg restricts enable_plugins).
