# Release Update (RU) Procedure

Apply a quarterly Oracle **Release Update** to a single-instance database on Oracle
19c/21c. This is the core patching procedure; RAC, Data Guard, and OS-specific variants
build on it in their own runbooks.

> **Applies to:** Oracle 19c / 21c single-instance. For RAC use **RAC Patch
> Procedure.md**; for standby use **Data Guard Patch Sequence.md**. Placeholders
> throughout; no real environment referenced.

---

## Prerequisites

- [ ] **Pre-Patch Checklist.md** completed (backup, ORACLE_HOME copy, guaranteed restore
      point, conflict check all done).
- [ ] **OPatch Upgrade Procedure.md** completed (OPatch ≥ RU minimum).
- [ ] RU patch staged, unzipped, checksum verified.

---

## 1. Final conflict check (against the staged RU)

```
cd <ru_patch_dir>
$ORACLE_HOME/OPatch/opatch prereq CheckConflictAgainstOHWithDetail -ph ./
```
Proceed only if it reports no conflicts.

## 2. Stop services using the home

```
# Stop the database and listener cleanly
sqlplus / as sysdba
SQL> SHUTDOWN IMMEDIATE;
SQL> EXIT;
lsnrctl stop <listener_name>
```
On Windows also stop the Oracle DB and listener **services** (see Windows Patch
Procedure.md). Confirm nothing else is using the home (`opatch lsinventory` should not be
blocked by a running process).

## 3. Apply the Release Update (binary patch)

From the patch directory, with the home offline:

```
cd <ru_patch_dir>
$ORACLE_HOME/OPatch/opatch apply
```

OPatch patches the Oracle binaries in the home. Watch for the **"OPatch succeeded"**
message; if it errors, stop and go to **Rollback Procedure.md** — do not start the
database on a half-patched home.

## 4. Restart and run datapatch (SQL changes)

The binary patch only updates files; **datapatch** applies the SQL/dictionary changes
inside the database.

```
lsnrctl start <listener_name>
sqlplus / as sysdba
SQL> STARTUP;
SQL> EXIT;

cd $ORACLE_HOME/OPatch
./datapatch -verbose
```

For a multitenant (CDB/PDB) database, `datapatch` automatically patches the root and all
open PDBs — ensure PDBs are OPEN first (`ALTER PLUGGABLE DATABASE ALL OPEN;`).

## 5. Recompile invalid objects

Patching can invalidate objects; recompile and re-check:

```
sqlplus / as sysdba
SQL> @?/rdbms/admin/utlrp.sql
SQL> SELECT COUNT(*) FROM dba_objects WHERE status='INVALID';
```
The count should return to (or below) the pre-patch baseline recorded in the checklist.

## 6. Confirm the patch is registered

```
$ORACLE_HOME/OPatch/opatch lspatches
sqlplus / as sysdba
SQL> SELECT patch_id, action, status, action_time
       FROM dba_registry_sqlpatch ORDER BY action_time;
```
The new RU should appear with `action='APPLY'` and `status='SUCCESS'`.

---

## Success criteria

- `opatch apply` reported success and `opatch lspatches` shows the new RU.
- `datapatch` completed; `dba_registry_sqlpatch` shows the RU APPLY/SUCCESS.
- Invalid-object count is at or below baseline.
- Then proceed to **Post-Patch Validation.md** for the full functional check.

## If anything fails

Stop and use **Rollback Procedure.md** (planned rollback) or **Emergency Rollback
Procedure.md** (production down). Never leave a database running on a partially applied
RU.
