# Oracle Patching Runbooks

Battle-tested **operational runbooks** for patching Oracle databases — quarterly Release
Updates, OPatch upgrades, and rollbacks — across single-instance, RAC, and Data Guard
configurations on both Windows and Linux. These are the step-by-step procedures a Senior
Oracle DBA follows to patch production safely and recover fast when something goes wrong.

Patching is where databases most often break, and recovery is the skill interviewers
probe hardest. This repository captures the full lifecycle: **prepare → upgrade OPatch →
apply → validate → roll back (planned or emergency)**, with the platform- and
architecture-specific variants that real environments require.

---

## Purpose

A Release Update is routine — until it isn't. The difference between a clean maintenance
window and a 3am outage is preparation and a rehearsed rollback. These runbooks encode
that discipline: every procedure starts from verified backups and a guaranteed restore
point, follows a precise apply sequence, validates with evidence, and has a defined path
back. They are written to be followed under pressure, by any DBA, without improvisation.

---

## Supported scope

| Dimension | Coverage |
|---|---|
| Oracle versions | **19c, 21c** (procedures apply to current Release Update model) |
| Architectures | Single-instance, **RAC** (rolling), **Data Guard** (standby-first) |
| Operating systems | **Linux** (Oracle Linux / RHEL), **Windows Server** |
| Patch types | Quarterly **Release Updates**, OPatch (6880880) upgrades, one-off patches |
| Recovery | Planned rollback (`opatch rollback` + `datapatch`, flashback) and emergency rollback |

---

## ⚠️ Disclaimer — sanitized & generic by design

All runbooks and sample outputs are **fully sanitized and generic**. They contain **no**
real hostnames, IPs, SIDs, service names, DBIDs, credentials, or employer/company data.
Patch IDs, paths (`/u01/app/oracle/...`), node names (`db-node-01`), and database names
(`ORADEMO`, `PDB1`) are **placeholders / fictional** examples. Patch numbers shown in
samples illustrate the format only.

These are reference procedures for portfolio and educational use. **Patching and rollback
are high-risk operations** — validate every step in a non-production environment and apply
under your own change-control process. Always read the specific patch README, which is
the authoritative source for that patch.

---

## Runbook index

Follow them in roughly this order; the platform/architecture runbooks layer on top of the
core sequence.

| Runbook | Use it to… |
|---|---|
| [Pre-Patch Checklist](runbooks/Pre-Patch%20Checklist.md) | Verify backups, restore point, conflicts, and readiness — the Go/No-Go gate |
| [OPatch Upgrade Procedure](runbooks/OPatch%20Upgrade%20Procedure.md) | Bring OPatch (6880880) up to the RU's minimum version |
| [Release Update Procedure](runbooks/Release%20Update%20Procedure.md) | Apply a quarterly RU to a single-instance database (the core flow) |
| [Post-Patch Validation](runbooks/Post-Patch%20Validation.md) | Prove, with evidence, that the patch registered and the DB is healthy |
| [Rollback Procedure](runbooks/Rollback%20Procedure.md) | Back a patch out cleanly while still in the window |
| [Data Guard Patch Sequence](runbooks/Data%20Guard%20Patch%20Sequence.md) | Patch primary/standby standby-first with no loss of DR |
| [RAC Patch Procedure](runbooks/RAC%20Patch%20Procedure.md) | Roll an RU through a cluster node by node with `opatchauto` |
| [Windows Patch Procedure](runbooks/Windows%20Patch%20Procedure.md) | Handle Windows services and file locks during a patch |
| [Linux Patch Procedure](runbooks/Linux%20Patch%20Procedure.md) | Handle Linux environment, ownership, and process checks |
| [Emergency Rollback Procedure](runbooks/Emergency%20Rollback%20Procedure.md) | Restore production fast when a patch causes an outage |

Sanitized, annotated example command output for the key steps is in
[`sample_outputs/`](sample_outputs/).

---

## How the runbooks fit together

```
                     ┌─────────────────────────┐
                     │   Pre-Patch Checklist    │  (Go / No-Go)
                     └────────────┬─────────────┘
                                  v
                     ┌─────────────────────────┐
                     │  OPatch Upgrade (if < min)│
                     └────────────┬─────────────┘
                                  v
        ┌──────────────── Release Update Procedure ────────────────┐
        │            (single-instance core sequence)               │
        │   layered by:  Linux / Windows  •  RAC  •  Data Guard     │
        └────────────┬───────────────────────────────┬────────────┘
                     v                                 v
        ┌─────────────────────────┐      ┌─────────────────────────┐
        │   Post-Patch Validation  │      │  Rollback / Emergency   │
        │      (sign-off)          │      │  Rollback (if failure)  │
        └─────────────────────────┘      └─────────────────────────┘
```

---

## Sample output

From `datapatch -verbose` (fictional patch IDs):

```
Patch 37501731 apply (pdb CDB$ROOT): SUCCESS
Patch 37501731 apply (pdb PDB$SEED): SUCCESS
Patch 37501731 apply (pdb PDB1):     SUCCESS
SQL Patching tool complete on Tue Jun 17 19:31:18 2026
```

From `dba_registry_sqlpatch` (fictional data):

```
  PATCH_ID TARGET_VERSION     ACTION     STATUS    ACTION_TIME
---------- ------------------ ---------- --------- ----------------
  37501731 21.15.0.0.0        APPLY      SUCCESS   2026-06-17 19:31
```

Each file in [`sample_outputs/`](sample_outputs/) ends with a **"Read:"** note explaining
what the output proves and what to do next.

---

## Repository structure

```
oracle-patching-runbooks/
├── README.md
├── LICENSE
├── .gitignore
├── runbooks/            # the 10 step-by-step patching/rollback procedures
├── sample_outputs/      # fictional, sanitized OPatch / datapatch / registry output
└── screenshots/         # (optional) terminal captures for the portfolio
```

---

## Core principles (applied throughout)

- **Never patch without a tested backup, an ORACLE_HOME copy, and a guaranteed restore
  point.** They are what make a fast rollback possible.
- **Binary and dictionary must stay consistent.** `opatch` patches the home; `datapatch`
  patches the database — both must match before "all clear."
- **Standby-first (Data Guard) and node-by-node (RAC)** preserve availability and always
  leave a working fallback.
- **Validate with evidence, not assumption** — `lspatches`, `dba_registry_sqlpatch`,
  invalid-object delta, and application sign-off.
- **In an outage, restore service first; reconcile binaries second.**

---

## Future enhancements

- **Quarterly patch calendar / tracker** mapping each environment to its current RU level.
- **`opatchauto` GI+RDBMS combined runbook** with worked multi-node timing.
- **Automated pre-check script** wrapping the checklist (backup/restore-point/conflict).
- **OJVM and Data Pump bundle** guidance alongside the database RU.
- **Grid Infrastructure-only patching** runbook (out-of-place / `gridSetup` patching).

---

## License

Released under the [MIT License](LICENSE) — free to use, adapt, and share with
attribution.
