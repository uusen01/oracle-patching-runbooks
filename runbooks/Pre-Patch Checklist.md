# Pre-Patch Checklist

Mandatory readiness checks to complete **before** applying any Oracle patch (Release
Update, one-off, or OJVM). The purpose is simple: never start a patch you cannot safely
finish or roll back. Skipping these is the most common cause of failed maintenance
windows.

> **Applies to:** Oracle 19c / 21c, single-instance and RAC, Windows and Linux.
> All paths, names, and IDs below are **placeholders** — substitute your own.
> Nothing here references any real or confidential environment.

---

## 1. Scope and approval

- [ ] Change record approved, maintenance window confirmed, stakeholders notified.
- [ ] Target patch identified (e.g., quarterly Release Update) and its README read end
      to end, including any post-install (`datapatch`) and known-issue notes.
- [ ] Rollback decision criteria agreed in advance (what failure triggers a rollback,
      and who authorizes it).

## 2. Backups and recoverability (non-negotiable)

- [ ] **Database backup** verified — a recent RMAN level-0 (or a guaranteed restore
      point) exists and was validated. Patching without a tested backup is not allowed.
- [ ] **ORACLE_HOME backup** planned — a copy/snapshot of the home before patching, so a
      binary rollback is possible even if `opatch rollback` fails:
      ```
      # Linux example (home offline):
      tar czf /backup/ohome_pre_<patch>.tgz -C $ORACLE_BASE/product/<ver> dbhome_1
      ```
- [ ] **Guaranteed restore point** created just before patching for a fast DB rollback:
      ```sql
      CREATE RESTORE POINT PRE_<PATCH_ID> GUARANTEE FLASHBACK DATABASE;
      ```
      (Requires ARCHIVELOG mode and sufficient FRA — verify both.)

## 3. Environment inventory (record current state)

- [ ] Current OPatch version meets the patch README minimum:
      `$ORACLE_HOME/OPatch/opatch version`
- [ ] Record current patch inventory (compare after):
      `$ORACLE_HOME/OPatch/opatch lspatches`
- [ ] Record `registry$history` / `dba_registry_sqlpatch` so you can confirm what
      changed afterward.
- [ ] Note ORACLE_HOME, ORACLE_SID(s), instance count, OS, and edition.

## 4. Conflict and prerequisite analysis

- [ ] Run the patch conflict check against the target home:
      ```
      $ORACLE_HOME/OPatch/opatch prereq CheckConflictAgainstOHWithDetail \
            -ph ./<patch_dir>
      ```
- [ ] Resolve any conflicts (obtain a merged/replacement patch from Oracle Support if
      needed). Do not proceed with unresolved conflicts.
- [ ] Confirm sufficient free space in ORACLE_HOME, the inventory, and the FRA.

## 5. Health snapshot (baseline to compare against)

- [ ] Capture a pre-patch health baseline: instance status, invalid objects count,
      tablespace usage, and (if licensed) an AWR snapshot. After patching you compare
      against this.
- [ ] `SELECT COUNT(*) FROM dba_objects WHERE status='INVALID';` — record the number.
- [ ] Confirm no active long-running jobs/backups will collide with the window.

## 6. Access and prerequisites in place

- [ ] OS and `oracle`/grid account credentials and `sudo`/Administrator access confirmed.
- [ ] Patch staged and unzipped in a clean directory; checksum verified.
- [ ] For RAC: confirm cluster health (`crsctl stat res -t`) and node order.
- [ ] For Data Guard: confirm standby is in sync and decide the patch order (see the
      Data Guard Patch Sequence runbook).

## 7. Communication plan

- [ ] Start/stop notifications ready; bridge/contact list for escalation.
- [ ] Application team aware of the outage window and ready to validate after.

---

## Go / No-Go

Proceed **only** when every item above is checked. If any backup, rollback, or conflict
item is incomplete, the window is **No-Go** — reschedule rather than risk an
unrecoverable failure.

Next: **OPatch Upgrade Procedure.md** (ensure OPatch is current) → **Release Update
Procedure.md** (apply) → **Post-Patch Validation.md** (verify).
