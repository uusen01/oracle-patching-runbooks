# Screenshots

Optional terminal captures used in the main README and for portfolio presentation.
Images are not required for the runbooks to be useful — they exist to show recruiters and
hiring managers real patching output at a glance.

## Capture guidance

- Run the commands against a **non-production / lab** database and capture the terminal
  output (dark theme, monospaced font, wide enough that output doesn't wrap).
- **Sanitize before capturing.** Confirm no real hostnames, IPs, service names, DBIDs,
  schema/user names, or company identifiers appear. Use a demo database (e.g., `ORADEMO`,
  node `db-node-01`) so visible names are generic, as in `sample_outputs/`.
- Save as PNG with a descriptive name, e.g. `opatch_apply.png`, `datapatch.png`.

## Recommended screenshots (highest portfolio signal first)

1. **`datapatch_success.png`** — `datapatch -verbose` finishing with SUCCESS per PDB.
   Shows you understand the binary-vs-SQL two-step that catches out junior DBAs.
2. **`opatch_apply.png`** — a clean "Patch NNNN successfully applied / OPatch succeeded."
3. **`dba_registry_sqlpatch.png`** — the patch history query proving registration.
4. **`opatchauto_rac.png`** — a node patched and rejoined ONLINE (RAC rolling).
5. **`flashback_rollback.png`** — emergency flashback to a restore point restoring
   service. Recovery confidence is a strong differentiator.

Five well-chosen, sanitized screenshots beat capturing everything. A captured
**before/after `opatch lspatches`** (e.g., 21.14 → 21.15) is a compact, convincing image
to add.
