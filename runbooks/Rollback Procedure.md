# Rollback Procedure (Planned)

How to cleanly back a Release Update or one-off patch out of a single-instance database
when post-patch validation fails but the situation is **controlled** (still inside the
maintenance window, no production outage). For a production-down scenario, use
**Emergency Rollback Procedure.md** instead.

> **Applies to:** Oracle 19c / 21c single-instance. RAC/standby rollbacks follow the same
> tools in the order described in their runbooks. Placeholders throughout; no real
> environment referenced.

---

## Decide the rollback method

There are two independent ways to undo a patch — pick based on what failed:

| Situation | Method |
|---|---|
| Patch applied but you want to remove it cleanly | **`opatch rollback` + `datapatch`** (Path A) |
| Database is corrupt/unstable or datapatch failed badly | **Guaranteed restore point Flashback** (Path B) |
| Binary rollback itself fails | **Restore the ORACLE_HOME backup** (Path C) |

Always prefer the least invasive path that fully resolves the problem.

---

## Path A — OPatch rollback (clean patch removal)

1. Identify the patch ID to remove:
   ```
   $ORACLE_HOME/OPatch/opatch lspatches
   ```
2. Stop the database and listener:
   ```sql
   SHUTDOWN IMMEDIATE;
   ```
   ```
   lsnrctl stop <listener_name>
   ```
3. Roll the binary patch out:
   ```
   $ORACLE_HOME/OPatch/opatch rollback -id <patch_id>
   ```
4. Restart and reverse the SQL changes with datapatch:
   ```
   sqlplus / as sysdba
   SQL> STARTUP;
   cd $ORACLE_HOME/OPatch
   ./datapatch -verbose          # detects the rollback and reverses SQL changes
   ```
5. Recompile and verify:
   ```sql
   @?/rdbms/admin/utlrp.sql
   SELECT patch_id, action, status FROM dba_registry_sqlpatch ORDER BY action_time;
   ```
   The patch should now show a `ROLLBACK` / `SUCCESS` row, and `opatch lspatches` should
   no longer list it.

---

## Path B — Flashback to the guaranteed restore point

Use when the database is unstable or datapatch left the dictionary inconsistent. This
reverts the **entire database** to the pre-patch point created in the checklist.

```sql
SHUTDOWN IMMEDIATE;
STARTUP MOUNT;
FLASHBACK DATABASE TO RESTORE POINT PRE_<PATCH_ID>;
ALTER DATABASE OPEN RESETLOGS;
```
> Flashback reverses the **database** but not the **binaries**. If the binary patch is
> still installed, also perform Path A (opatch rollback) or Path C so the home and the
> database are consistent. After OPEN RESETLOGS, take a fresh full backup.

---

## Path C — Restore the ORACLE_HOME backup

Use only if `opatch rollback` fails (corrupt inventory/home).

1. With the database and listener down, restore the pre-patch home copy taken in the
   checklist:
   ```
   # Linux example
   rm -rf $ORACLE_HOME
   tar xzf /backup/ohome_pre_<patch>.tgz -C $ORACLE_BASE/product/<ver>
   ```
2. Start the database from the restored home and run `datapatch -verbose` to align the
   dictionary with the (now un-patched) binaries.
3. Validate as in Path A.

---

## After any rollback

- [ ] Run **Post-Patch Validation.md** checks (they apply equally to a rolled-back state).
- [ ] Confirm invalid objects are back to baseline and applications work.
- [ ] Record what failed and why in the change record; capture logs for the Oracle SR.
- [ ] Decide remediation: obtain a merged/replacement patch, then reschedule.

---

## Key principles

- **Never** leave a database running on a partially applied or partially rolled-back
  patch. Binary state and dictionary state must match.
- Binaries and database are rolled back with **different** tools — `opatch` handles the
  home, `datapatch`/flashback handle the database. Keep them consistent.
- The guaranteed restore point and ORACLE_HOME backup are what make a clean rollback
  possible — which is why the checklist makes them mandatory.
