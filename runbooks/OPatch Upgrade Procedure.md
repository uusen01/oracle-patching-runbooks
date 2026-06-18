# OPatch Upgrade Procedure

How to update the **OPatch utility** itself to the version required by a Release Update.
Most RU failures that happen "before the patch even starts" are caused by an OPatch that
is too old. This is a short, standalone procedure run before applying any RU.

> **Applies to:** Oracle 19c / 21c, Windows and Linux, single-instance and RAC (repeat
> per home / per node). Placeholders throughout; no real environment referenced.

---

## Why this matters

Each Release Update README states a **minimum OPatch version**. OPatch is shipped
separately from the database and updated frequently (patch 6880880). Applying an RU with
an outdated OPatch produces prerequisite or apply failures, so confirming/upgrading
OPatch is always step one of the apply.

---

## 1. Check the current version

```
$ORACLE_HOME/OPatch/opatch version
```

Compare against the **minimum OPatch version** in the target RU README. If current ≥
required, skip to validation; otherwise upgrade.

## 2. Obtain the correct OPatch

- Download **patch 6880880** for the matching database release (e.g., 19c or 21c) and
  platform from Oracle Support. There is a different OPatch zip per major release/platform.
- Stage it in a clean directory and verify the checksum.

## 3. Back up the existing OPatch directory

```
# Linux
mv $ORACLE_HOME/OPatch $ORACLE_HOME/OPatch_backup_$(date +%F)
```
```
:: Windows (PowerShell)
Rename-Item "%ORACLE_HOME%\OPatch" "OPatch_backup"
```
Keeping the old directory lets you revert instantly if the new OPatch misbehaves.

## 4. Install the new OPatch

Unzip the 6880880 archive directly into ORACLE_HOME — it creates a fresh `OPatch`
directory:

```
# Linux
unzip -q p6880880_<ver>_<platform>.zip -d $ORACLE_HOME
```
```
:: Windows
Expand-Archive p6880880_<ver>_MSWIN-x86-64.zip -DestinationPath "%ORACLE_HOME%"
```

For RAC, repeat on **every** node so OPatch versions match cluster-wide.

## 5. Validate

```
$ORACLE_HOME/OPatch/opatch version          # confirm new version >= required
$ORACLE_HOME/OPatch/opatch lsinventory       # confirm inventory is readable
```

Both must succeed. `lsinventory` failing here usually means an inventory pointer or
permissions problem — resolve before attempting the RU.

---

## Rollback (if the new OPatch causes problems)

```
# Linux
rm -rf $ORACLE_HOME/OPatch
mv $ORACLE_HOME/OPatch_backup_<date> $ORACLE_HOME/OPatch
```

Because you backed up the original directory in step 3, reverting is immediate and
risk-free.

---

## Notes

- OPatch upgrades touch only the utility, not the database — no datapatch or DB restart
  is required for the OPatch upgrade itself.
- Always upgrade OPatch in the **same home** you are about to patch; a multi-home server
  may need it done per home.

Next: **Release Update Procedure.md**.
