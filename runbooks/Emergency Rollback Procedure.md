# Emergency Rollback Procedure (Production Down)

The fast path when a patch has left **production down or severely degraded** and you must
restore service immediately. This prioritizes **time to recovery** over a tidy patch
removal. For a controlled, in-window rollback, use **Rollback Procedure.md** instead.

> **Applies to:** Oracle 19c / 21c. Use during an active incident. Placeholders
> throughout; no real environment referenced. Engage Oracle Support (Sev 1) in parallel.

---

## First 5 minutes — stabilize and decide

1. **Declare an incident.** Notify stakeholders and the on-call chain; start an incident
   log (timestamped actions).
2. **Stop making it worse.** Do not run more ad-hoc changes. Capture the alert log and
   the failing `opatch`/`datapatch` output for the SR before you change state.
3. **Pick the fastest safe recovery** based on symptom:

| Symptom | Fastest recovery |
|---|---|
| DB open but patch caused bad behavior; binaries OK | **Flashback to restore point** (Path 1) |
| `datapatch`/dictionary broken; DB won't open cleanly | **Flashback to restore point** (Path 1) |
| Binary apply failed; home corrupt; DB won't start | **Restore ORACLE_HOME backup** (Path 2) |
| RAC: one node bad, others serving | **Evict/patch-rollback the bad node** (Path 3) |

The guaranteed restore point and ORACLE_HOME backup from the Pre-Patch Checklist are what
make sub-15-minute recovery possible — this is why they are mandatory.

---

## Path 1 — Flashback Database to the pre-patch restore point (fastest)

Reverts the entire database to the moment before patching.

```sql
SHUTDOWN ABORT;            -- speed over cleanliness during an outage
STARTUP MOUNT;
FLASHBACK DATABASE TO RESTORE POINT PRE_<PATCH_ID>;
ALTER DATABASE OPEN RESETLOGS;
```
- Service is restored at the pre-patch state in minutes.
- The **binaries may still be patched** — if so, the home and DB are inconsistent. Once
  service is stable, schedule **opatch rollback** (Rollback Procedure.md Path A) or home
  restore so binary and dictionary levels match.
- Take a **fresh full backup** immediately after OPEN RESETLOGS.

## Path 2 — Restore the ORACLE_HOME backup

Use when the binary apply corrupted the home or the database cannot start at all.

```
# Database and listener down
# Linux example:
rm -rf $ORACLE_HOME
tar xzf /backup/ohome_pre_<patch>.tgz -C $ORACLE_BASE/product/<ver>
```
```
:: Windows: stop the Oracle services first, then restore the pre-patch home copy
```
Then start the instance from the restored home and run `datapatch -verbose` to align the
dictionary with the un-patched binaries. If the database content itself is also bad,
combine with Path 1 (flashback) or an RMAN restore.

## Path 3 — RAC: isolate the failed node

If one node failed mid-rolling-patch but others still serve:

```
# Keep the cluster serving on healthy nodes; on the bad node (root):
$GRID_HOME/OPatch/opatchauto rollback <ru_patch_dir>
# If the node won't stabilize, stop its stack so it stops affecting the cluster:
crsctl stop crs
```
Restore service on healthy nodes first, then recover the failed node out of the critical
path. Do not run `datapatch` until the cluster binary level is consistent again.

## Path 4 — Last resort: RMAN restore

If flashback is unavailable (no restore point / flashback off) and the home restore is
not enough, perform a full RMAN restore/recover to the pre-patch time. This is the
slowest option — see the RMAN repo's Restore and Recovery Guide.

---

## After service is restored

- [ ] Confirm the database is open and applications work (quick smoke test).
- [ ] Ensure **binary and dictionary levels are consistent** (don't leave a flashed-back
      DB on patched binaries).
- [ ] Take a fresh full backup.
- [ ] Keep all logs; update the Oracle SR with the failure evidence.
- [ ] Hold a blameless post-incident review: what failed, why the pre-checks didn't catch
      it, and what to change before re-attempting the patch.

---

## Key principles

- **Restore service first, tidy up second.** In an outage, `SHUTDOWN ABORT` + flashback is
  acceptable; reconciling binaries can follow once users are back.
- **Consistency is mandatory before "all clear."** Binary level and dictionary level must
  match before you call the incident resolved.
- **The restore point + home backup are the lifeline.** Every minute of recovery time you
  save here was bought during the Pre-Patch Checklist.
