# Reference — Known Issues

Agent reference for version-specific known issues in Talos Linux. Consult the relevant version file before performing upgrades, troubleshooting unexpected behavior, or recommending configurations.

---

## How to Use

1. **Identify the Talos version** the user is running: `talosctl version -n <node>`
2. **Open the corresponding file** (e.g., `talos-1.9.md` for Talos 1.9.x)
3. **Match the symptom** described by the user against documented issues
4. **Apply the documented workaround** if one exists, or note the affected version range and suggest an upgrade

---

## File Naming Convention

Each file is named `talos-X.Y.md` where X.Y is the Talos minor release series:

```
known-issues/
  talos-1.8.md   # Issues specific to Talos 1.8.x
  talos-1.9.md   # Issues specific to Talos 1.9.x
```

---

## Issue Entry Format

Every issue follows this structure:

```markdown
### <Short descriptive title>

**Symptom:** What the user observes (error message, unexpected behavior).

**Cause:** Root cause explanation.

**Workaround:** Step-by-step fix or configuration change.

**Affected versions:** Which patch releases are affected (e.g., 1.9.0-1.9.2).

**Status:** One of:
- `fixed in X.Y.Z` — resolved in a specific release
- `workaround available` — not fixed upstream, but a config change avoids it
- `won't fix` — by design or out of scope
- `investigating` — upstream issue open, no resolution yet
```

---

## Version Files

- [talos-1.8.md](./talos-1.8.md) — Talos 1.8.x known issues
- [talos-1.9.md](./talos-1.9.md) — Talos 1.9.x known issues
