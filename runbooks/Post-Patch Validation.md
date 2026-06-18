# Post-Patch Validation

The verification performed **after** a patch is applied and the database is back up. The
goal is to prove — with evidence — that the patch registered correctly, the database is
healthy, and applications work, before declaring the maintenance window successful.

> **Applies to:** Oracle 19c / 21c, single-instance and RAC, Windows and Linux.
> Placeholders throughout; no real environment referenced.

---

## 1. Patch registration (binary + SQL)

- [ ] Binary patch present:
      ```
      $ORACLE_HOME/OPatch/opatch lspatches
      ```
- [ ] SQL patch applied successfully:
      ```sql
      SELECT patch_id, version, action, status, action_time
        FROM dba_registry_sqlpatch
       ORDER BY action_time;
      ```
      The new RU must show `action='APPLY'`, `status='SUCCESS'`.
- [ ] `datapatch -verbose` output reviewed — no errors, all PDBs patched (CDB).

## 2. Database availability and state

- [ ] Instance OPEN, correct OPEN_MODE and ARCHIVELOG:
      ```sql
      SELECT status FROM v$instance;
      SELECT open_mode, log_mode FROM v$database;
      ```
- [ ] For CDB: all PDBs OPEN (no RESTRICTED):
      ```sql
      SELECT name, open_mode, restricted FROM v$pdbs;
      ```
- [ ] Listener up and services registered:
      ```
      lsnrctl status <listener_name>
      ```

## 3. Invalid objects (compare to baseline)

- [ ] Recompile and compare to the pre-patch baseline from the checklist:
      ```sql
      @?/rdbms/admin/utlrp.sql
      SELECT COUNT(*) FROM dba_objects WHERE status='INVALID';
      ```
      Should be at or below baseline. Investigate any *new* invalids that won't recompile.

## 4. Component and registry health

- [ ] Database registry components VALID:
      ```sql
      SELECT comp_name, version, status FROM dba_registry ORDER BY comp_name;
      ```
- [ ] No new corruption / alert-log errors since restart — scan the alert log for
      `ORA-` errors during and after the patch.

## 5. RAC-specific (if applicable)

- [ ] All instances up and patch level consistent across nodes:
      ```
      crsctl stat res -t
      $ORACLE_HOME/OPatch/opatch lspatches   # run on each node; versions must match
      ```
- [ ] Services relocated/running on the expected nodes (`srvctl status service -d <db>`).

## 6. Data Guard-specific (if applicable)

- [ ] Standby applying redo, no gap:
      ```sql
      SELECT name, value FROM v$dataguard_stats WHERE name LIKE '%lag%';
      ```
- [ ] Primary and standby at the **same** patch level (see Data Guard Patch Sequence.md).

## 7. Performance sanity

- [ ] Spot-check key queries / a representative workload; compare against the pre-patch
      baseline (and an AWR snapshot if licensed). Watch for plan regressions — a patch can
      change optimizer behavior.

## 8. Application validation

- [ ] Application/owner team runs their smoke tests and confirms functional success.
      The window is not "done" until the application owner signs off.

## 9. Close-out

- [ ] Record results in the change record: patch ID, datapatch status, invalid-object
      delta, any issues and resolutions, and total downtime.
- [ ] **Keep the guaranteed restore point and ORACLE_HOME backup** until the database has
      run cleanly through at least one full business cycle, then drop the restore point:
      ```sql
      DROP RESTORE POINT PRE_<PATCH_ID>;
      ```

---

## Sign-off

Declare success only when sections 1–8 pass and the application owner confirms.
Otherwise treat it as a failed patch and consult **Rollback Procedure.md**.
