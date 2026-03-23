---
name: ruflo-init
description: >
  Safely install Ruflo (claude-flow) for the first time into a Claude Code project
  that may already have existing .claude/settings.json, CLAUDE.md, .claude/agents/,
  .claude/commands/, or .claude/helpers/ configuration. Use this skill whenever the
  user wants to run `ruflo init`, `claude-flow init`, or add Ruflo to a project for
  the first time — especially when they have an existing Claude Code setup and are
  worried about overwriting their config. Triggers on "add ruflo to my project",
  "initialize ruflo here", "set up ruflo". Do NOT use for upgrading an existing
  Ruflo installation — use the ruflo-upgrade skill instead. Never skip this skill
  because the user seems experienced; the merge step requires judgment every time.
---

You are executing the `ruflo-init` skill. Follow every step exactly. Never run Ruflo directly in the live project directory — always stage first. Do not write any live files until the user explicitly approves the plan.

---

## Step 1 — Prerequisites check

Run:
```bash
node --version 2>/dev/null | grep -E "v(1[89]|[2-9][0-9])" || \
  { echo "ERROR: Node 18+ required"; exit 1; }
```

If Node 18+ is not found: stop immediately. Tell the user to install Node 18+ from https://nodejs.org and re-run the skill.

---

## Step 2 — Inventory existing config

Run and capture all output:
```bash
echo "=== settings.json ===" && \
  (cat .claude/settings.json 2>/dev/null || echo "(not found)")
echo "=== settings.local.json ===" && \
  (cat .claude/settings.local.json 2>/dev/null || echo "(not found)")
echo "=== CLAUDE.md ===" && \
  (cat CLAUDE.md 2>/dev/null || echo "(not found)")
echo "=== agents ===" && \
  (ls .claude/agents/ 2>/dev/null || echo "(not found)")
echo "=== commands ===" && \
  (ls .claude/commands/ 2>/dev/null || echo "(not found)")
echo "=== helpers ===" && \
  (ls .claude/helpers/ 2>/dev/null || echo "(not found)")
echo "=== .mcp.json ===" && \
  (cat .mcp.json 2>/dev/null || echo "(not found)")
echo "=== claude-flow.config.json ===" && \
  (cat claude-flow.config.json 2>/dev/null || echo "(not found)")
echo "=== existing snapshot ===" && \
  (cat .claude/ruflo-snapshot/manifest.json 2>/dev/null || echo "(not found)")
```

Report exactly what was found.

**Fast path — nothing exists**: If every section above showed "(not found)", skip directly to Step 4. Tell the user: "No existing Claude config found — running Ruflo init directly." No backup or merge is needed.

**Existing snapshot detected**: If `.claude/ruflo-snapshot/manifest.json` exists, warn the user:
> "A ruflo-snapshot already exists, suggesting Ruflo was previously initialized here. Did you mean to use the ruflo-upgrade skill instead?"

Offer two options: (a) continue with re-init, or (b) stop and run ruflo-upgrade. Wait for the user's choice before continuing.

---

## Step 3 — Backup existing files

If any config existed in Step 2, create a timestamped backup:

```bash
BACKUP_DIR=".claude/backup-$(date +%Y%m%d-%H%M%S)"
mkdir -p "$BACKUP_DIR/agents" "$BACKUP_DIR/commands" "$BACKUP_DIR/helpers"

[ -f .claude/settings.json ]       && cp .claude/settings.json "$BACKUP_DIR/settings.json"
[ -f .claude/settings.local.json ] && cp .claude/settings.local.json "$BACKUP_DIR/settings.local.json"
[ -f CLAUDE.md ]                   && cp CLAUDE.md "$BACKUP_DIR/CLAUDE.md"
[ -d .claude/agents ]              && cp .claude/agents/*.md "$BACKUP_DIR/agents/" 2>/dev/null
[ -d .claude/commands ]            && cp -r .claude/commands/. "$BACKUP_DIR/commands/" 2>/dev/null
[ -d .claude/helpers ]             && cp -r .claude/helpers/. "$BACKUP_DIR/helpers/" 2>/dev/null
[ -f .mcp.json ]                   && cp .mcp.json "$BACKUP_DIR/.mcp.json"
[ -f claude-flow.config.json ]     && cp claude-flow.config.json "$BACKUP_DIR/claude-flow.config.json"

echo "Backed up to $BACKUP_DIR"
ls -la "$BACKUP_DIR"
```

Confirm the backup succeeded. If any file failed to copy, stop and report. Record the exact `$BACKUP_DIR` path — you will print it in the final confirmation.

---

## Step 4 — Stage Ruflo init output

**Never run Ruflo in the live project directory.** Run it in a temp directory:

```bash
STAGING_DIR=$(mktemp -d)
trap 'rm -rf "$STAGING_DIR"' EXIT

mkdir -p "$STAGING_DIR/.claude/agents" \
         "$STAGING_DIR/.claude/commands" \
         "$STAGING_DIR/.claude/helpers"

(cd "$STAGING_DIR" && npx ruflo@latest init --wizard 2>&1)
INIT_EXIT=$?

echo "Init exit code: $INIT_EXIT"
ls -la "$STAGING_DIR/.claude/" 2>/dev/null
```

**If init exits non-zero**: Stop. Show the full stderr output. Remind the user their originals are safe in the backup. Do not proceed.

**If init exits 0 but wrote no files**: Report: "Init appeared to succeed but wrote nothing to the staging directory. Consider running `npx ruflo@latest init --force` manually." Do not proceed.

**Fallback**: If `ruflo@latest` fails with a package-not-found error, retry with:
```bash
(cd "$STAGING_DIR" && npx claude-flow@latest init --wizard 2>&1)
```
Note which package was used in all subsequent output.

**Helper discovery**: After successful staging, scan hook commands in the staged `settings.json` for referenced file paths (strings matching `.claude/`, `$CLAUDE_PROJECT_DIR`, or paths ending in `.cjs`, `.mjs`, `.sh`, `.js`). For each path found, check whether the file exists in `$STAGING_DIR`. Report missing helpers as warnings (non-blocking).

---

## Step 5 — Diff and classify

**Fresh project (nothing in Step 2)**: All staged files are new. Mark every file "New from Ruflo — add as-is" and skip to Step 6.

**Prior config existed**: Compare OURS (backup) vs THEIRS (staging) for each file in the managed set:

- `.claude/settings.json`
- `CLAUDE.md`
- `.claude/agents/*.md`
- `.claude/commands/*`
- `.claude/helpers/*` (all files)
- `.claude/statusline.sh` (if present)

Classification:
- **New**: exists in staging, not in backup → add as-is
- **Unchanged**: exists in both, identical (`diff -q`) → skip
- **Modified**: exists in both, different → apply merge rules (see Step 6)
- **Removed**: exists in backup, not in staging → keep user's version, warn

Present a summary table:
```
File                          | Status    | Action
------------------------------|-----------|--------------------------------
.claude/settings.json         | Modified  | Merge (JSON rules)
CLAUDE.md                     | Modified  | Append Ruflo section
.claude/agents/coder.md       | New       | Add as-is
.claude/helpers/hook-handler  | New       | Add as-is
```

---

## Step 6 — Propose merged files

For each Modified file, compute a proposed merge.

### `settings.json` (ownership: `shared`)

Merge at the JSON key level — never by line.

**`hooks` array — semantic merge:**
- Hook identity key: `(event, normalizedCommand)` — normalize by stripping `$CLAUDE_PROJECT_DIR` vs relative path variants, collapsing whitespace.
- Same identity in both → deduplicate (keep one)
- Only in user's version → keep
- Only in Ruflo's version → add
- Same identity, different body → surface as conflict
- Order: user hooks first, Ruflo hooks appended.

**`permissions` / `allowedTools`:**
- Union of both sets. Never remove a permission the user had.

**All other top-level keys:**
- Prefer user's value. If Ruflo's value differs, flag it in the proposal summary but do not block.

**Unknown keys**: Never delete a key not present in Ruflo's output.

Show the full proposed JSON in a code block. Below it, list every merge decision:
```
Merge decisions for settings.json:
  + Added 5 Ruflo hooks (PostToolUse×3, PreToolUse×2)
  ✓ Kept your 2 existing hooks
  ✓ Kept your permissions (unioned with Ruflo's)
  ⚠ Hook event "SubagentEnd" unrecognised by this skill's validator — kept but flagging
  ⚠ Helper .claude/helpers/router.sh referenced but not found in staging
```

**Soft validation** — run after computing the merge, surface as warnings (never block):
- JSON syntactically valid: `node -e "JSON.parse(require('fs').readFileSync('.claude/settings.json.ruflo-tmp','utf8'))"`
- Hook event names recognised by this skill's validator (valid: `PreToolUse`, `PostToolUse`, `PostToolUseFailure`, `Stop`, `SubagentStop`, `SubagentStart`, `Notification`, `UserPromptSubmit`, `SessionStart`, `SessionEnd`, `PreCompact`, `PermissionRequest`) — warn on any others as "unrecognised by this skill's validator" (not necessarily invalid)
- All file paths in hook commands exist on disk
- No duplicate hooks after normalisation
- Permission patterns use `:*` suffix for prefix matching (warn on bare `*`)

### `CLAUDE.md` (ownership: `section-managed`)

Check for `<!-- ruflo:start -->` markers in the existing file:

- **Markers absent**: Keep the entire existing file untouched. Append a new Ruflo block at the bottom:
  ```markdown
  <!-- ruflo:start version=VERSION -->
  [Ruflo content from staging]
  <!-- ruflo:end -->
  ```
- **Markers present**: Replace only the content between `<!-- ruflo:start ... -->` and `<!-- ruflo:end -->`. Never touch content outside the markers.
- **Markers appear more than once**: Fall into conflict mode — show both occurrences and ask the user to resolve manually before proceeding.

Show a preview of the proposed final `CLAUDE.md` with the insertion point indicated.

### Agent, command, and helper files (ownership: `whole-file`)

- **New files**: List them — no user review needed.
- **Modified files** (re-init only): Show a diff. Ask: (1) keep mine, (2) take Ruflo's, (3) keep both (rename Ruflo's to `<name>.ruflo.md`).

---

## Step 7 — Two-tier approval

### Tier 1: Global plan

Present the full plan before writing anything:

```
Here's my proposed plan:

Auto-accept (new files, no conflict):
  ✅ .claude/agents/coder.md       — new from Ruflo
  ✅ .claude/helpers/hook-handler.cjs — new from Ruflo
  ✅ .claude/commands/sparc.md     — new from Ruflo

Merged (requires your review):
  📝 .claude/settings.json         — your 2 hooks kept + 5 Ruflo hooks added
  📝 CLAUDE.md                     — your content preserved, Ruflo block appended

Validation warnings (non-blocking):
  ⚠ Hook event "SubagentEnd" unrecognised by this skill's validator
  ⚠ .claude/helpers/router.sh referenced but not generated

Your originals are safe at: .claude/backup-YYYYMMDD-HHMMSS/
```

Ask: **"Does this look right? Say 'apply' to proceed, or tell me what to change."**

**Do NOT write any live files until the user approves.** Accept "apply", "yes", "looks good", "go ahead", or similar.

### Tier 2: Per-conflict decisions (only if genuine conflicts exist)

For each conflict, show both versions and collect the user's choice before applying anything:

```
CONFLICT: .claude/agents/coder.md

--- YOUR VERSION ---
[content]

--- RUFLO VERSION ---
[content]

Options: (1) keep mine  (2) take Ruflo's  (3) keep both (rename Ruflo's to coder.ruflo.md)
```

---

## Step 8 — Atomic apply

Once user approves, apply atomically — never write directly to live paths mid-apply.

**Step 8a — Write to temp paths:**
```bash
# Write each resolved file to a .ruflo-tmp path
cat resolved-settings.json > .claude/settings.json.ruflo-tmp
cat resolved-CLAUDE.md > CLAUDE.md.ruflo-tmp
# etc. for each modified file
```

**Step 8b — Validate:**
```bash
node -e "JSON.parse(require('fs').readFileSync('.claude/settings.json.ruflo-tmp','utf8'))" \
  && echo "valid" || echo "INVALID — aborting"
```

If validation fails: delete all `.ruflo-tmp` files, report the error. Live files are untouched. Stop.

**Step 8c — Rename to final paths:**
```bash
mv .claude/settings.json.ruflo-tmp .claude/settings.json
mv CLAUDE.md.ruflo-tmp CLAUDE.md
# etc.
```

If any rename fails mid-apply, report which files succeeded and which did not.

**Step 8d — Copy new files from staging:**
```bash
cp "$STAGING_DIR/.claude/agents/coder.md" .claude/agents/coder.md
# etc. for each new agent, command, helper
```

**Step 8e — Soft validation on final files.** Run the same checks as Step 6 against the written files. Report as warnings, do not block.

---

## Step 9 — Save snapshot

After successful apply, write the snapshot. The snapshot is the merge base for future upgrades — it records exactly what Ruflo wrote, not what the user had before.

```bash
RUFLO_VERSION=$(npx ruflo@latest --version 2>/dev/null | tr -d '\n')
SNAPSHOT_DIR=".claude/ruflo-snapshot"
mkdir -p "$SNAPSHOT_DIR/agents" "$SNAPSHOT_DIR/commands" "$SNAPSHOT_DIR/helpers"

# shared: copy pure Ruflo output from staging, NOT the merged live file.
# This preserves Ruflo's unmodified output as the merge base for future upgrades.
cp "$STAGING_DIR/.claude/settings.json" "$SNAPSHOT_DIR/settings.json"

# section-managed: extract only the Ruflo sentinel block from staging.
# The snapshot must not include user content outside the markers.
sed -n '/<!-- ruflo:start/,/<!-- ruflo:end -->/p' \
  "$STAGING_DIR/CLAUDE.md" > "$SNAPSHOT_DIR/CLAUDE.md"

# whole-file: copy from the live applied file (correct — these are entirely Ruflo-owned).
[ -d .claude/agents ]   && cp .claude/agents/*.md "$SNAPSHOT_DIR/agents/" 2>/dev/null
[ -d .claude/commands ] && cp -r .claude/commands/. "$SNAPSHOT_DIR/commands/" 2>/dev/null
[ -d .claude/helpers ]  && cp -r .claude/helpers/. "$SNAPSHOT_DIR/helpers/" 2>/dev/null
```

Compute SHA-256 hashes (macOS compatible):
```bash
shasum -a 256 <file> | awk '{print $1}'
```

Write `manifest.json` with every managed file listed:
```json
{
  "rufloVersion": "<version>",
  "timestamp": "<ISO-8601-UTC>",
  "initCommand": "npx ruflo@latest init --wizard",
  "files": {
    ".claude/settings.json": { "hash": "<sha256>", "ownership": "shared",
      "_note": "hash of pure Ruflo staging output, not the merged file" },
    "CLAUDE.md":             { "hash": "<sha256>", "ownership": "section-managed",
      "_note": "hash of Ruflo sentinel block only, not the full merged file" },
    ".claude/agents/coder.md": { "hash": "<sha256>", "ownership": "whole-file" },
    ".claude/helpers/hook-handler.cjs": { "hash": "<sha256>", "ownership": "whole-file" }
  }
}
```

Ownership rules:
- `.claude/settings.json` → `"shared"` — snapshot from `$STAGING_DIR`
- `CLAUDE.md` → `"section-managed"` — snapshot is sentinel block only, from `$STAGING_DIR`
- Everything else → `"whole-file"` — snapshot from live applied file

---

## Step 10 — `.gitignore` offer

Check whether the snapshot and backup paths are already ignored:
```bash
grep -q "ruflo-snapshot" .gitignore 2>/dev/null && echo "already ignored" || echo "not ignored"
```

If not already present, offer:
> "Would you like me to add `.claude/ruflo-snapshot/` and `.claude/backup-*/` to `.gitignore`? These are machine-local files and generally should not be committed."

If the user agrees:
```bash
echo "" >> .gitignore
echo "# Ruflo skill artifacts" >> .gitignore
echo ".claude/ruflo-snapshot/" >> .gitignore
echo ".claude/backup-*/" >> .gitignore
```

---

## Step 11 — Final confirmation

Print a full summary:

```
✅ Ruflo initialized successfully.

Written:
  .claude/settings.json       (merged: your 2 hooks + 5 Ruflo hooks)
  CLAUDE.md                   (your content preserved, Ruflo block appended)
  .claude/agents/coder.md     (new)
  .claude/agents/researcher.md (new)
  .claude/helpers/hook-handler.cjs (new)
  .claude/commands/sparc.md   (new)

Snapshot saved: .claude/ruflo-snapshot/ (ruflo <version>)
Originals at:   .claude/backup-YYYYMMDD-HHMMSS/
Restore with:
  cp .claude/backup-YYYYMMDD-HHMMSS/settings.json .claude/settings.json
  cp .claude/backup-YYYYMMDD-HHMMSS/CLAUDE.md CLAUDE.md

Validation warnings:
  ⚠ Hook event "SubagentEnd" unrecognised by this skill's validator — kept but verify it works

Next: add Ruflo as an MCP server for runtime use:
  claude mcp add ruflo -- npx -y ruflo@latest mcp start
```

---

## Edge cases

**Init silently writes nothing**: Detect with `ls -la "$STAGING_DIR/.claude/"`. Report and stop. Suggest `npx ruflo@latest init --force`.

**Empty or minimal existing files**: Still run the full merge flow. An empty file is still intentional state.

**Node not available**: Report error with https://nodejs.org link, stop immediately.

**Staging dir cleanup**: The `trap 'rm -rf "$STAGING_DIR"' EXIT` set in Step 4 handles cleanup on both success and failure.

**Package not found**: If both `ruflo@latest` and `claude-flow@latest` fail, stop and tell the user to check their npm registry access.
