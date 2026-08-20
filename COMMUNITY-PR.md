# Update: Template Proxmox VE REST API

Update of the existing template in `Virtualization/template_proxmox-ve-rest-api-zabbix/7.0/`.

Template UUID and template name are unchanged and the existing object UUIDs are preserved, so
this is an in-place update: existing installations keep their item history when importing with
**Update existing**. The template group changes from `Templates` to `Templates/Virtualization`
to follow the official guideline that requires a category after the `Templates/` prefix.

| | before | after |
|---|---|---|
| Items | 31 | 47 |
| Discovery rules | 7 | 10 |
| Item prototypes | 52 | 68 |
| Triggers | 11 | 21 |
| Macros | 16 | 26 |
| API endpoints | 8 | 16 |

## Cluster support

This is the main change. Guest discovery previously read `/nodes/{node}/qemu` and
`/nodes/{node}/lxc`, so only guests on the single node given by `{$PVE_NODE}` were monitored,
and a live migration made every item of the migrated guest unsupported.

- `discover.qemu` and `discover.lxc` are now dependent on `/cluster/resources`. VMs and
  containers on every node of the cluster are discovered from one Zabbix host, and migrations
  no longer break their items.
- New LLD macros `{#NODE}` and `{#TEMPLATE}`.
- New items `vm.node[{#VMID}]` and `lxc.node[{#VMID}]` report the node a guest currently runs
  on, with an informational trigger that fires when the value changes, which indicates a
  migration.
- Five guest values are not part of `/cluster/resources` and are still read from the node given
  by `{$PVE_NODE}`: QEMU `balloon`, `balloon_min` and `running-machine`, LXC `swap` and
  `maxswap`. For guests on other nodes they stay empty instead of turning unsupported.
- Node-scoped data (disks, host interfaces, storage, tasks, time, version, host status) is still
  read per node. Monitoring several nodes in that depth means one Zabbix host per PVE node.

## New monitoring areas

Three new discovery rules and sixteen new items:

| Area | Source | Contents |
|---|---|---|
| Physical disks | `/nodes/{node}/disks/list` | SMART health, size, SSD wearout, with triggers on health and wearout |
| Host network interfaces | `/nodes/{node}/network` | active state, autostart, type, interface-down trigger |
| HA | `/cluster/ha/status/current` | CRM manager status, per-resource state, error-state trigger |
| Cluster status | `/cluster/status` | cluster name, quorum, nodes online/offline/total, guests running/total |
| Node time | `/nodes/{node}/time` | local time and timezone |

Two calculated items were added so that percentage values exist as items instead of only inside
trigger expressions: `pve.memory.util` and `pve.rootfs.util`.

Standalone installations are handled explicitly: `pve.cluster.name` reports `standalone`,
`pve.cluster.quorum` reports `1`, and the HA discovery finds nothing without turning
unsupported, so none of these produce false alarms on a single node.

The interface-down trigger requires the interface to have been active before and skips
interfaces without `autostart`, and `{$IFACE.NOT_MATCHES}` drops the per-guest interfaces that
Proxmox creates and destroys together with a VM or container.

## Rebuilt dashboard

The dashboard was rebuilt around the widgets introduced in Zabbix 7.0 and now uses widget
communication, so the previous 22 graph-prototype widgets are replaced by five pages that can be
navigated interactively.

| Page | Contents |
|---|---|
| Overview | six Item value tiles, three Gauges (CPU, memory, root filesystem) with 80/90 thresholds, SVG graph, two Pie charts, Problems |
| Virtual machines | Item navigator grouped by the `vmid` tag, Honeycomb of VM states, plus Item value and Graph that follow the selection |
| LXC containers | the same for containers |
| Storage and disks | Honeycomb of storage utilization and disk health, navigator and Graph |
| Nodes and HA | Honeycomb of node states and HA resource states, four Item value tiles |

The drill-down uses the documented data source mechanism: `itemnavigator` and `honeycomb`
broadcast `_itemid`, and `item`, `gauge` and `graph` consume it through a
`<field>._reference` field.

## Consistent item tags

Every item and item prototype now carries a component tag and an instance tag. Previously eight
master items had no tags at all, and five discovery rules produced items without any instance
tag, so filtering by a single storage, node, user or task was impossible.

- New uniform instance tag `vmid` on all guest, task and backup items, which makes it possible to
  select one guest and see all of its values, and which is what the dashboard groups by.
- Instance tags added for storage, nodes, users, backups and tasks.
- The stray `User` tag namespace was folded into `PVE`.
- Master items are now tagged `PVE: Raw`.

## Bug fixes

- **HA manager status never returned data.** The master item read `/cluster/ha/status`, which is
  only a directory index and carries no status. Every poll fell through to the error handler.
  On a production host the item had written 1429 consecutive fallback values. The source is now
  `/cluster/ha/status/current`, and the manager status is taken from the entry with
  `type=master`.
- **Item key typos**, UUIDs kept so history survives the rename:
  `vm.baloon` to `vm.balloon`, `vm.baloonmin` to `vm.balloonmin`,
  `user.exouration` to `user.expiration`.
- **Macro typos**: `{$ENABLE_BACKUP_ALER}`, `{$ENABLE_STORAGE_AVAILABLE_ALER}` and
  `{$ENABLE_TASK_ALER}` now end in `ALERT`.
- `vm.vcpu` was missing `.first()` on its JSONPath and relied on trimming the array brackets
  instead.
- Hardcoded storage thresholds of 90 and 95 percent in trigger names and expressions replaced by
  `{$STORAGE.UTIL.WARN}` and `{$STORAGE.UTIL.CRIT}`.
- `opdata` fields shortened to stay within the 255 character limit.
- The file in this repository is named `template_proxmox-ve-rest-api.yaml.yaml` with a duplicated
  extension. This pull request renames it to `template_proxmox-ve-rest-api.yaml`.

## Reliability

Error handlers on JSONPath steps went from 13 of 82 to 59 of 98. This matters because in the
Proxmox API schema for `/nodes/{node}/qemu` only `vmid` and `status` are non-optional; every
other field may be absent. Previously a guest that disappeared between a discovery run and the
next poll, or a single missing optional field, turned items unsupported. Now such a value is
discarded, or replaced by the LLD macro for name items, and the item stays supported.

## New macros

`{$CLUSTER.NODES.OFFLINE.MAX}`, `{$DISK.WEAROUT.MIN}`, `{$IFACE.ACTIVE.WINDOW}`,
`{$IFACE.NOT_MATCHES}`, `{$LXC.CPU.WARN}`, `{$LXC.CPU.HIGH}`, `{$MEMORY.UTIL.MAX}`,
`{$PVE.USER.EXPIRE.TIME}`, `{$ROOTFS.UTIL.WARN}`, `{$ROOTFS.UTIL.CRIT}`,
`{$STORAGE.UTIL.WARN}`, `{$STORAGE.UTIL.CRIT}`, plus the three renamed `ENABLE_*` macros.

## Verification

- Every endpoint and every JSON field used by the template was checked against the official
  Proxmox VE API schema (`api-viewer/apidoc.js`, 452 endpoints).
- All JSONPath expressions of the new guest discovery were executed against live
  `/cluster/resources` output of a production Proxmox VE host with 20 VMs and one container.
- All dashboard widget field names and the data source syntax were checked against the Zabbix
  7.0 frontend sources, including the widget manifests that declare which widget broadcasts and
  accepts which type.
- The template was imported into Zabbix 7.0.29 and has no unsupported items on that host.
- Structural checks: 195 UUIDs, all valid version 4, no duplicates, no duplicate item tags,
  maximum dependent-item nesting depth 1, every trigger, graph, calculated item and dashboard
  reference resolves to an existing item key.
