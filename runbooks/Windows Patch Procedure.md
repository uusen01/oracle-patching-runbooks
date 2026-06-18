# Windows Patch Procedure

Platform-specific steps for applying an Oracle Release Update on **Windows Server**. The
patch logic is identical to **Release Update Procedure.md**; this runbook covers the
Windows differences — services, file locks, and Administrator context — that trip up
DBAs used to Linux.

> **Applies to:** Oracle 19c / 21c on Windows Server, single-instance. Placeholders
> throughout; no real environment referenced.

---

## Windows-specific considerations

| Difference | What to do |
|---|---|
| Database runs as a **Windows service** | Stop the service, not just `SHUTDOWN`, so files unlock |
| File locking is stricter | All processes using ORACLE_HOME must be stopped or `opatch apply` fails on locked DLLs |
| Run as **Administrator** | Open the command prompt "Run as administrator"; the OS user must own the home |
| Paths use backslashes / `%ORACLE_HOME%` | Adjust commands accordingly |

---

## 1. Pre-checks

Complete **Pre-Patch Checklist.md** and **OPatch Upgrade Procedure.md** (Windows OPatch
zip). Additionally:

- [ ] Identify the exact service names:
      ```
      sc query | findstr /i Oracle
      ```
      (Typically `OracleService<SID>` and `OracleOraDB<ver>HomeTNSListener`.)
- [ ] Confirm you are in an **elevated** (Administrator) command prompt.

## 2. Stop the database cleanly, then the services

```
sqlplus / as sysdba
SQL> SHUTDOWN IMMEDIATE;
SQL> EXIT;
```
Then stop the Windows services so the binaries unlock:
```
net stop OracleService<SID>
net stop Oracle<ver>TNSListener
```
Also close any SQL*Plus / SQL Developer / agent sessions pointed at the home — a single
open handle on a DLL will block the apply.

## 3. Apply the Release Update

From an elevated prompt, in the patch directory:
```
cd <ru_patch_dir>
%ORACLE_HOME%\OPatch\opatch apply
```
If OPatch reports **files in use / locked**, find and stop the holding process (Resource
Monitor or `Get-Process`), then re-run. Do not force-delete locked files.

## 4. Start services and run datapatch

```
net start OracleService<SID>
net start Oracle<ver>TNSListener

sqlplus / as sysdba
SQL> STARTUP;        -- if the service start did not already open it
SQL> EXIT;

cd %ORACLE_HOME%\OPatch
datapatch -verbose
```

## 5. Recompile and validate

```
sqlplus / as sysdba
SQL> @?\rdbms\admin\utlrp.sql
SQL> SELECT COUNT(*) FROM dba_objects WHERE status='INVALID';
```
Then run the full **Post-Patch Validation.md**.

---

## Common Windows pitfalls

- **"opatch apply fails on a locked DLL"** — a process still has the home open. Stop the
  services and any client tools; check Resource Monitor's "Associated Handles."
- **Service won't stop** — a long-running query or backup may be active; let it finish or
  stop it gracefully rather than killing the service.
- **Permission denied** — the command prompt is not elevated, or the OS user is not the
  home owner. Re-open as Administrator.
- **Antivirus locking files** — exclude ORACLE_HOME from real-time scanning during the
  window if AV interferes with the apply.

## Rollback

Use **Rollback Procedure.md** / **Emergency Rollback Procedure.md**; the only Windows
delta is stopping the services (as above) before `opatch rollback` and restarting them
after.
