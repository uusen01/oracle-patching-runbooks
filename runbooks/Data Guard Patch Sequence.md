# Data Guard Patch Sequence

The correct order for applying a Release Update to a primary/physical-standby Data Guard
configuration with **minimal downtime and no loss of DR protection**. The principle:
patch the standby first, verify, then switch over and patch the old primary — so a
working, patched database is always available.

> **Applies to:** Oracle 19c / 21c physical standby (single-instance or RAC standby).
> Combine with **RAC Patch Procedure.md** when nodes are clustered. Placeholders
> throughout; no real environment referenced.

---

## Concept: standby-first, rolling

Because the standby is a separate home, you can patch it while the primary keeps serving
users. A brief role switch then lets you patch the former primary. Redo apply is
coordinated so the databases are never left at incompatible patch levels longer than
necessary.

```
[Primary: serving]        [Standby: APPLY]
      |  1. patch standby home + datapatch on standby (after switchover)   |
      |  2. switchover  -> standby becomes primary (patched)               |
      |  3. patch old primary (now standby)                                |
      |  4. switch back (optional)                                         |
```

---

## 0. Pre-checks (in addition to Pre-Patch Checklist.md)

- [ ] Standby in sync, no gap:
      ```sql
      SELECT name, value FROM v$dataguard_stats WHERE name LIKE '%lag%';
      ```
- [ ] Data Guard Broker state healthy (if used): `DGMGRL> SHOW CONFIGURATION;`
- [ ] Same OPatch version staged on **both** homes; RU staged on both.
- [ ] Backups and (on the primary) a guaranteed restore point in place.

## 1. Stop redo apply on the standby

```sql
-- On standby
ALTER DATABASE RECOVER MANAGED STANDBY DATABASE CANCEL;
```
Or, with Broker: `DGMGRL> EDIT DATABASE '<standby>' SET STATE='APPLY-OFF';`

## 2. Patch the standby ORACLE_HOME (binary only)

On the standby, shut the instance and apply the RU binaries exactly as in **Release
Update Procedure.md** steps 2–3 (`opatch apply`). **Do not** run datapatch on the standby
— a physical standby is in managed recovery; SQL changes arrive via redo from the primary
after switchover. Restart the standby in MOUNT and re-enable apply:

```sql
STARTUP MOUNT;
ALTER DATABASE RECOVER MANAGED STANDBY DATABASE DISCONNECT FROM SESSION;
```
Let it catch up; confirm zero gap again.

## 3. Switch over (standby becomes primary)

During the agreed window, switch roles so the patched database takes over:

```
DGMGRL> SWITCHOVER TO '<standby_db>';
```
Users now run on the **patched** former-standby. Run `datapatch -verbose` on the **new
primary** so the SQL changes are applied and propagate via redo.

## 4. Patch the former primary (now standby)

Repeat step 2 on the old primary (now the standby): stop apply, `opatch apply` the RU
binaries, restart MOUNT, re-enable managed recovery. Both homes are now at the same
binary level and the dictionary change has flowed through redo.

## 5. (Optional) switch back

If you require the original primary site as primary, switch over again once both sides
are patched and in sync.

## 6. Validate

- [ ] Both databases at the same patch level (`opatch lspatches` on each).
- [ ] `dba_registry_sqlpatch` shows the RU SUCCESS on the primary.
- [ ] Redo transport/apply healthy, zero gap (`v$dataguard_stats`).
- [ ] Run **Post-Patch Validation.md** on the primary; application sign-off.

---

## Key principles

- **Patch the standby first** so DR protection and a fallback primary always exist.
- **datapatch runs on the primary only** — never on a physical standby in managed
  recovery; the SQL changes replicate via redo.
- Keep the gap at zero before each role transition; never switch over with a lag.
- If anything fails on the standby, the un-patched primary is still serving — you have a
  safe fallback, which is the whole point of the standby-first order.
