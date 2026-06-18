# RAC Patch Procedure (Rolling)

Apply a Release Update to an Oracle RAC cluster **node by node** so the database stays
available throughout — a rolling patch. While one node is patched, the others keep
serving connections.

> **Applies to:** Oracle 19c / 21c RAC (Grid Infrastructure + RDBMS homes). Assumes the
> RU is rolling-installable (the README states this). Placeholders throughout; no real
> environment referenced.

---

## Concept: one node at a time

```
Node 1 (patch)   Node 2 (serving)   Node 3 (serving)
   -> patched, rejoin
                 Node 2 (patch)   Node 3 (serving)
                    -> patched, rejoin
                                  Node 3 (patch) -> patched, rejoin
```
Connections drain off the node being patched (via services/FAN) and run on the remaining
nodes, so the database never fully stops.

---

## 0. Pre-checks (plus Pre-Patch Checklist.md)

- [ ] Cluster healthy on all nodes:
      ```
      crsctl stat res -t
      crsctl check cluster -all
      ```
- [ ] RU confirmed **rolling** in its README; OPatch ≥ minimum on every node.
- [ ] Same RU staged on every node; backups + guaranteed restore point in place.
- [ ] Note the GI and RDBMS home paths and the node order.

## 1. Use opatchauto (recommended for GI + RDBMS together)

`opatchauto` patches both the Grid Infrastructure and database homes on a node and
handles stack stop/start. Run it **as root** per node, one node at a time:

```
# On node 1 (root):
$GRID_HOME/OPatch/opatchauto apply <ru_patch_dir>
```
It stops the local stack, patches GI + RDBMS, restarts, and runs node-level steps. Wait
for success before touching the next node.

## 2. Verify the node rejoined cleanly

```
crsctl stat res -t                 # all local resources ONLINE
$ORACLE_HOME/OPatch/opatch lspatches
srvctl status database -d <db>     # instance back up on this node
```
Confirm services relocated back as expected and connections are balanced.

## 3. Repeat for each remaining node

Move to node 2, then node 3, etc. — **never** patch two nodes at once in a rolling
operation. The cluster runs at mixed patch levels only transiently, which RAC supports
for a rolling RU.

## 4. Run datapatch ONCE, after the last node

The SQL/dictionary change is applied to the database a single time, after all nodes have
the binaries:

```
# From any one node, once:
cd $ORACLE_HOME/OPatch
./datapatch -verbose
```
For CDB, ensure PDBs are OPEN so they are patched too.

## 5. Validate the cluster

- [ ] `opatch lspatches` shows the **same** RU on every node.
- [ ] `dba_registry_sqlpatch` shows the RU APPLY/SUCCESS (once).
- [ ] `crsctl stat res -t` all ONLINE; services on expected nodes.
- [ ] Invalid objects at/below baseline; run **Post-Patch Validation.md**.

---

## Manual alternative (if not using opatchauto)

Per node, in order: drain/relocate services → `srvctl stop instance -d <db> -n <node>` →
stop the local GI stack (`crsctl stop crs`) → `opatch apply` GI and RDBMS homes →
restart stack and instance → verify → next node. Then `datapatch` once at the end.
`opatchauto` automates exactly this and is less error-prone.

---

## Rollback

Rolling rollback mirrors the apply: `opatchauto rollback <patch_dir>` node by node, then
`datapatch` to reverse SQL changes once. See **Rollback Procedure.md** for detail and the
database-level (flashback) fallback.

## Key principles

- **One node at a time** preserves availability; never patch the whole cluster at once in
  a rolling op.
- **datapatch runs once**, after the last node — not per node.
- Verify each node rejoins ONLINE before proceeding; a node that fails to rejoin is an
  immediate stop-and-assess.
