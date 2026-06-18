# Linux Patch Procedure

Platform-specific steps for applying an Oracle Release Update on **Linux** (Oracle
Linux / RHEL). The patch logic is identical to **Release Update Procedure.md**; this
runbook covers the Linux specifics — environment, ownership, and process checks.

> **Applies to:** Oracle 19c / 21c on Linux, single-instance. For clustered Linux use
> **RAC Patch Procedure.md**. Placeholders throughout; no real environment referenced.

---

## Linux-specific considerations

| Topic | Note |
|---|---|
| OS user | Run as the software owner (commonly `oracle`); GI steps may need `grid`/`root` |
| Environment | Set `ORACLE_HOME`, `ORACLE_SID`, `PATH` correctly (e.g., via `oraenv`) |
| Process check | Ensure no process is using the home before `opatch apply` |
| Permissions | The OS user must own the home and inventory; avoid running RDBMS patches as root |

---

## 1. Pre-checks

Complete **Pre-Patch Checklist.md** and **OPatch Upgrade Procedure.md** (Linux OPatch
zip). Additionally:

- [ ] Set the environment for the target home:
      ```bash
      . oraenv            # enter the SID, or export ORACLE_HOME / ORACLE_SID / PATH
      echo $ORACLE_HOME; echo $ORACLE_SID
      ```
- [ ] Confirm you are the correct OS user:
      ```bash
      id                  # expect the software owner, e.g. oracle
      ```
- [ ] Confirm no stray processes hold the home:
      ```bash
      ps -ef | grep -i pmon          # see running instances
      fuser $ORACLE_HOME/bin/oracle  # who is using the binary
      ```

## 2. Stop the database and listener cleanly

```bash
sqlplus / as sysdba <<'EOF'
SHUTDOWN IMMEDIATE;
EXIT;
EOF
lsnrctl stop <listener_name>
```
Confirm the instance and listener are down (`ps -ef | grep pmon`, `lsnrctl status`).

## 3. Apply the Release Update

```bash
cd <ru_patch_dir>
$ORACLE_HOME/OPatch/opatch apply
```
Watch for **"OPatch succeeded."** If it errors, stop and go to **Rollback Procedure.md** —
do not start the database on a half-patched home.

## 4. Restart and run datapatch

```bash
lsnrctl start <listener_name>
sqlplus / as sysdba <<'EOF'
STARTUP;
EXIT;
EOF

cd $ORACLE_HOME/OPatch
./datapatch -verbose
```
For CDB, ensure PDBs are OPEN (`ALTER PLUGGABLE DATABASE ALL OPEN;`) before datapatch.

## 5. Recompile and validate

```bash
sqlplus / as sysdba <<'EOF'
@?/rdbms/admin/utlrp.sql
SELECT COUNT(*) FROM dba_objects WHERE status='INVALID';
EXIT;
EOF
```
Then run the full **Post-Patch Validation.md**.

---

## Common Linux pitfalls

- **"OUI-67073 / inventory problems"** — wrong OS user or inventory pointer
  (`/etc/oraInst.loc`); run as the home owner.
- **`opatch apply` hangs** — a background process (agent, leftover instance) still holds
  the home; identify with `fuser`/`ps` and stop it.
- **Space errors mid-apply** — insufficient free space in ORACLE_HOME or the inventory;
  the checklist's space check prevents this.
- **`datapatch` reports PDBs skipped** — those PDBs were not OPEN; open them and re-run.

## Rollback

Use **Rollback Procedure.md** (planned) or **Emergency Rollback Procedure.md** (production
down). On Linux the only platform note is running rollback steps as the correct OS user.
