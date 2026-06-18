# Repository Setup Guide (publishing checklist)

> For you, Usen — NOT portfolio content. After you set the GitHub topics and confirm
> everything looks right, delete it (or keep it private). It explains where every file
> belongs, the topics to add, the screenshots to capture, and how to publish
> `oracle-patching-runbooks`.

---

## 1. Where every file belongs

```
oracle-patching-runbooks/                    <- repository root
├── README.md                                <- main landing page
├── LICENSE                                   <- MIT license (GitHub auto-detects at root)
├── .gitignore                               <- keeps logs, patch zips, credentials out
├── REPO_SETUP.md                             <- THIS file (delete before/after publishing)
│
├── runbooks/                                 <- the 10 procedures
│   ├── Pre-Patch Checklist.md
│   ├── OPatch Upgrade Procedure.md
│   ├── Release Update Procedure.md
│   ├── Post-Patch Validation.md
│   ├── Rollback Procedure.md
│   ├── Data Guard Patch Sequence.md
│   ├── RAC Patch Procedure.md
│   ├── Windows Patch Procedure.md
│   ├── Linux Patch Procedure.md
│   └── Emergency Rollback Procedure.md
│
├── sample_outputs/                           <- fictional, sanitized command output
│   ├── opatch_version_and_lspatches.txt
│   ├── opatch_prereq_conflict_check.txt
│   ├── opatch_apply.txt
│   ├── datapatch_verbose.txt
│   ├── dba_registry_sqlpatch.txt
│   ├── opatchauto_rac_apply.txt
│   ├── opatch_rollback.txt
│   └── flashback_emergency_rollback.txt
│
└── screenshots/
    └── README.md
```

Rationale: `README.md` and `LICENSE` at the root so GitHub renders/detects them; the
procedures under `runbooks/` (these ARE the deliverable and render as markdown on GitHub);
fictional command output under `sample_outputs/` so a reviewer sees real OPatch/datapatch
results without a lab.

---

## 2. GitHub topics (add via the ⚙️ next to "About")

```
oracle
oracle-database
patching
release-update
opatch
datapatch
dba
database-administration
oracle-19c
oracle-21c
rac
data-guard
runbook
high-availability
database-engineering
```

Suggested **About** description:
> *Operational runbooks for patching Oracle 19c/21c — Release Updates, OPatch, RAC
> rolling, Data Guard standby-first, Windows/Linux, and planned + emergency rollback.
> Sanitized.*

---

## 3. Screenshots to capture (optional, high-impact)

From a lab DB, sanitized, dark terminal. Priority order:

1. `datapatch_success.png` — datapatch SUCCESS per PDB
2. `opatch_apply.png` — "Patch NNNN successfully applied"
3. `dba_registry_sqlpatch.png` — patch history query
4. `opatchauto_rac.png` — node patched + rejoined ONLINE
5. `flashback_rollback.png` — emergency flashback restoring service

See `screenshots/README.md`. A before/after `opatch lspatches` image is a compact win.

---

## 4. Publishing steps

```bash
cd oracle-patching-runbooks
git init
git add .
git commit -m "Oracle patching runbooks: RU, OPatch, RAC, Data Guard, rollback (19c/21c)"
git branch -M main
git remote add origin https://github.com/uusen01/oracle-patching-runbooks.git
git push -u origin main
```

Then on github.com/uusen01/oracle-patching-runbooks:
1. Add the **topics** from section 2 and set the **About** description.
2. Confirm `LICENSE` shows "MIT" in the sidebar.
3. (Optional) add screenshots and link them in the README.
4. **Pin** the repo on your profile.
5. Ensure the profile README "Featured" link points here as a clickable markdown link.

---

## 5. Final sanitization pass before pushing

- [ ] No real hostnames, IPs, SIDs, service names, or DBIDs (placeholders only;
      `ORADEMO`, `PDB1`, `db-node-01`, patch IDs are fictional/illustrative).
- [ ] No credentials; no `tnsnames.ora`/`login.sql`/password files committed (.gitignore).
- [ ] No Oracle patch binaries (`p*.zip`) or real OPatch logs committed (.gitignore).
- [ ] No USPS / AT&T / GM / employer identifiers in any file.
- [ ] Runbooks state they are generic references and to follow the actual patch README +
      change control.
- [ ] Delete this `REPO_SETUP.md` if you don't want it public.

---

## Note on RAC / Data Guard runbooks

Your hands-on RAC and Data Guard experience comes from the AT&T and GM roles (per the
Master Experience Document), not USPS — so these runbooks are well within what you can
speak to in an interview. They are written generically (no employer reference) and
demonstrate exactly the rolling-patch and standby-first expertise that $135k–$170k
postings ask for.
