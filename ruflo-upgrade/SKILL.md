---
name: ruflo-upgrade
description: >
  Safely upgrade an existing Ruflo (claude-flow) installation in a Claude Code
  project. Uses 3-way merge logic to distinguish Ruflo-to-Ruflo changes (applied
  automatically with bulk confirmation) from user customisations (always surfaced
  explicitly). Use whenever the user wants to upgrade Ruflo, says "update ruflo",
  "ruflo is outdated", "bump ruflo", or "run ruflo upgrade". Do NOT use for fresh
  installs — use ruflo-init instead. If no snapshot exists, falls back to a safe
  conservative mode that asks about every change.
---

You are executing the `ruflo-upgrade` skill. Follow every step exactly. Never run Ruflo in the live project directory — always stage first. Do not write any live files until the user explicitly approves the plan.

---

## Step 1 — Prerequisites and version check

```bash
node --version 2>/dev/null | grep -E "v(1[89]|[2-9][0-9])" || \
  { echo "ERROR: Node 18+ required"; exit 1; }
```

If Node check fails: stop, tell user to install Node 18+ from https://nodejs.org.

Check installed vs latest version:
```bash
CURRENT_VERSION=$(cat .claude/ruflo-snapshot/manifest.json 2>/dev/null | \
  node -e "const d=require('fs').readFileSync('/dev/stdin','utf8'); \
           console.log(JSON.parse(d).rufloVersion)" 2>/dev/null || echo "unknown")

LATEST_VERSION=$(npx ruflo@latest --version 2>/dev/null | tr -d '\n')

echo "Installed (snapshot): $CURRENT_VERSION"
echo "Latest available:     $LATEST_VERSION"
```

**Versions match**: Tell the user Ruflo is already up to date. Stop unless they explicitly ask to force a re-run.

**No snapshot found**: Warn the user:
> "No ruflo-snapshot found. I can't automatically distinguish your customisations from Ruflo-owned content. I'll treat every changed file as a potential user change and ask about everything. To avoid this on future upgrades, use ruflo-init first."

Offer: (a) continue in conservative mode, or (b) stop. Wait for the user's choice. If they choose to continue, set `CONSERVATIVE_MODE=1` and carry that flag through all subsequent steps.

---

## Step 2 — Backup current state

Back up everything before touching anything, including the current snapshot:

```bash
BACKUP_DIR=".claude/backup-$(date +%Y%m%d-%H%M%S)"
mkdir -p "$BACKUP_DIR/agents" "$BACKUP_DIR/commands" "$BACKUP_DIR/helpers" \
         "$BACKUP_DIR/ruflo-snapshot"

[ -f .claude/settings.json ]       && cp .claude/settings.json "$BACKUP_DIR/"
[ -f .claude/settings.local.json ] && cp .claude/settings.local.json "$BACKUP_DIR/"
[ -f CLAUDE.md ]                   && cp CLAUDE.md "$BACKUP_DIR/"
[ -d .claude/agents ]              && cp .claude/agents/*.md "$BACKUP_DIR/agents/" 2>/dev/null
[ -d .claude/commands ]            && cp -r .claude/commands/. "$BACKUP_DIR/commands/" 2>/dev/null
[ -d .claude/helpers ]             && cp -r .claude/helpers/. "$BACKUP_DIR/helpers/" 2>/dev/null
[ -f .mcp.json ]                   && cp .mcp.json "$BACKUP_DIR/"
[ -f claude-flow.config.json ]     && cp claude-flow.config.json "$BACKUP_DIR/"
[ -d .claude/ruflo-snapshot ]      && cp -r .claude/ruflo-snapshot/. \
                                       "$BACKUP_DIR/ruflo-snapshot/" 2>/dev/null

echo "Backed up to $BACKUP_DIR"
ls -la "$BACKUP_DIR"
```

Record the exact `$BACKUP_DIR` path — print it in every subsequent confirmation. If any copy fails, stop and report.

---

## Step 3 — Stage Ruflo upgrade output

**Never run Ruflo in the live project directory.**

```bash
STAGING_DIR=$(mktemp -d)
trap 'rm -rf "$STAGING_DIR"' EXIT

mkdir -p "$STAGING_DIR/.claude/agents" \
         "$STAGING_DIR/.claude/commands" \
         "$STAGING_DIR/.claude/helpers"

# Try native upgrade command first
(cd "$STAGING_DIR" && npx ruflo@latest init upgrade 2>&1)
UPGRADE_EXIT=$?

# Fall back to wizard if upgrade subcommand is not available
if [ $UPGRADE_EXIT -ne 0 ]; then
  echo "init upgrade not available — falling back to init --wizard"
  (cd "$STAGING_DIR" && npx ruflo@latest init --wizard 2>&1)
  UPGRADE_EXIT=$?
  USED_FALLBACK=1
fi

echo "Upgrade exit code: $UPGRADE_EXIT"
ls -la "$STAGING_DIR/.claude/"
```

Note: `init upgrade` likely implements skip-if-exists, not 3-way merging. Running it in a clean staging dir is intentional — the staging dir has no pre-existing files, so we get a pure "what new Ruflo would write from scratch" output that we then 3-way merge ourselves.

If upgrade exits non-zero: stop immediately. Show the full error. The live project is untouched.

If fallback was used (`USED_FALLBACK=1`), note this in the final confirmation.

**Helper discovery**: Scan staged `settings.json` hook commands for referenced file paths (`.claude/`, `$CLAUDE_PROJECT_DIR`, paths ending in `.cjs`, `.mjs`, `.sh`, `.js`). Verify those helpers exist in the staging dir. Report missing ones as warnings.

---

## Step 4 — 3-way classification

For every file in the managed set, load three versions:

```
BASE   = .claude/ruflo-snapshot/<file>    (what old Ruflo wrote)
OURS   = current project <file>           (may have user edits on top of BASE)
THEIRS = $STAGING_DIR/.claude/<file>      (what new Ruflo wants)
```

Run diffs:
```bash
# BASE→OURS: did the user edit this file?
diff -q ".claude/ruflo-snapshot/<file>" ".claude/<file>" \
  > /dev/null 2>&1 && USER_EDITED=0 || USER_EDITED=1

# BASE→THEIRS: did new Ruflo update this file?
diff -q ".claude/ruflo-snapshot/<file>" "$STAGING_DIR/.claude/<file>" \
  > /dev/null 2>&1 && RUFLO_UPDATED=0 || RUFLO_UPDATED=1
```

**Special case — `CLAUDE.md`**: Do not diff the whole file. The snapshot stores only the Ruflo sentinel block; OURS contains the full file including user content outside the markers. Diff only the sentinel block contents:

```bash
# Extract the sentinel block from OURS
sed -n '/<!-- ruflo:start/,/<!-- ruflo:end -->/p' CLAUDE.md > /tmp/ours-ruflo-block.txt

# BASE is already sentinel-block-only (stored that way in the snapshot)
# THEIRS: extract sentinel block from staged output
sed -n '/<!-- ruflo:start/,/<!-- ruflo:end -->/p' \
  "$STAGING_DIR/CLAUDE.md" > /tmp/theirs-ruflo-block.txt

diff -q ".claude/ruflo-snapshot/CLAUDE.md" /tmp/ours-ruflo-block.txt \
  > /dev/null 2>&1 && USER_EDITED_BLOCK=0 || USER_EDITED_BLOCK=1

diff -q ".claude/ruflo-snapshot/CLAUDE.md" /tmp/theirs-ruflo-block.txt \
  > /dev/null 2>&1 && RUFLO_UPDATED_BLOCK=0 || RUFLO_UPDATED_BLOCK=1
```

User content outside the sentinel markers never contributes to conflict classification. A conflict is only raised when both the user edited inside the block AND Ruflo changed the block. If the user only edited outside the markers, `CLAUDE.md` is classified as "User edit only" (outside scope) and the Ruflo block is updated freely.

Classification table:

| BASE→OURS | BASE→THEIRS | Classification | Default action |
|---|---|---|---|
| No | No | Unchanged | Skip silently |
| No | Yes | Ruflo update | Auto-apply (bulk confirm) |
| Yes | No | User edit only | Keep as-is |
| Yes | Yes, same result | Both identical | Auto-apply |
| Yes | Yes, different | Genuine conflict | Always ask user |
| BASE absent | Any | No baseline | Ask user (conservative) |

**Conservative mode** (no snapshot): Treat every changed file as "Potential conflict — ask user." Present every diff and ask individually.

Present the full classification table:
```
File                          | Classification   | Action
------------------------------|------------------|---------------------------
.claude/settings.json         | Ruflo update     | Auto-apply (confirm)
CLAUDE.md                     | Genuine conflict | Ask user
.claude/agents/coder.md       | User edit only   | Keep yours, no change
.claude/agents/researcher.md  | Ruflo update     | Auto-apply (confirm)
.claude/agents/security.md    | New from Ruflo   | Add automatically
.claude/helpers/hook-handler  | Ruflo update     | Auto-apply (confirm)
.claude/commands/sparc.md     | Unchanged        | Skip
```

---

## Step 5 — Produce upgrade plan (two-tier approval)

### Tier 1: Global plan

Group by action and present before writing anything:

```
Ruflo updates — no user edits detected (auto-apply with one confirmation):
  .claude/settings.json        — 3 new hooks added, 1 hook command path updated
  .claude/agents/researcher.md — updated by Ruflo
  .claude/agents/security.md   — new agent added
  .claude/helpers/hook-handler.cjs — helper updated

User edits only — Ruflo unchanged (keeping yours):
  .claude/agents/coder.md      — your customisations untouched

Unchanged — skip:
  .claude/commands/sparc.md

Genuine conflicts — need your decision:
  CLAUDE.md                    — you edited inside the Ruflo block AND Ruflo updated it

Your pre-upgrade files are at: .claude/backup-YYYYMMDD-HHMMSS/
```

For the auto-apply group, show the BASE→THEIRS diff for each file, then ask once:
> "These files were updated by Ruflo with no user edits detected. Apply all automatically? (yes / show me each one first)"

Default is yes.

**Do NOT write any live files until the user approves.**

### Tier 2: Per-conflict decisions

For each genuine conflict, show a 3-way view and wait for the user's decision before moving on:

```
CONFLICT: CLAUDE.md (inside ruflo-managed section)

--- YOUR VERSION (inside <!-- ruflo:start --> block) ---
[user edits]

--- NEW RUFLO VERSION ---
[new Ruflo content]

--- BASELINE (what old Ruflo wrote) ---
[original Ruflo content]

My proposed merge: [show auto-merge proposal if possible]

Options: (a) accept my proposal  (b) keep yours  (c) take Ruflo's  (d) show full diff
```

**For `settings.json` conflicts**: Apply the same JSON merge rules as ruflo-init — semantic hook dedup on `(event, normalizedCommand)`, permission union, user-value preference for other keys. The 3-way classification informs which parts are user-owned vs Ruflo-owned.

Collect all conflict decisions before applying anything.

---

## Step 6 — Atomic apply

Apply all resolved files atomically — never write directly to live paths mid-apply.

**Step 6a — Write to temp paths:**
```bash
cat resolved-settings.json > .claude/settings.json.ruflo-tmp
cat resolved-CLAUDE.md > CLAUDE.md.ruflo-tmp
# etc. for each modified file
```

**Step 6b — Validate:**
```bash
node -e "JSON.parse(require('fs').readFileSync('.claude/settings.json.ruflo-tmp','utf8'))" \
  && echo "valid" || echo "INVALID — aborting"
```

If validation fails: delete all `.ruflo-tmp` files, report the error. Live files are untouched. Stop.

**Step 6c — Rename to final paths:**
```bash
mv .claude/settings.json.ruflo-tmp .claude/settings.json
mv CLAUDE.md.ruflo-tmp CLAUDE.md
# etc.
```

If any rename fails mid-apply, report which files succeeded and which did not.

**Step 6d — Copy new files from staging:**
```bash
cp "$STAGING_DIR/.claude/agents/security.md" .claude/agents/security.md
# etc. for each new agent, command, helper
```

**Step 6e — Soft validation** on final `settings.json`. Run all checks from Step 4 of the init skill:
- JSON syntax valid
- Hook event names recognised by this skill's validator (`PreToolUse`, `PostToolUse`, `PostToolUseFailure`, `Stop`, `SubagentStop`, `SubagentStart`, `Notification`, `UserPromptSubmit`, `SessionStart`, `SessionEnd`, `PreCompact`, `PermissionRequest`) — warn on any others as "unrecognised by this skill's validator" (not necessarily invalid)
- All file paths in hook commands exist on disk
- No duplicate hooks after normalisation
- Permission patterns use `:*` suffix

Report as warnings, do not block.

---

## Step 7 — Update snapshot

After successful apply, update the snapshot to record the new Ruflo version as the new merge base.

```bash
NEW_VERSION=$(npx ruflo@latest --version 2>/dev/null | tr -d '\n')
SNAPSHOT_DIR=".claude/ruflo-snapshot"
mkdir -p "$SNAPSHOT_DIR/agents" "$SNAPSHOT_DIR/commands" "$SNAPSHOT_DIR/helpers"

# shared: copy pure Ruflo output from staging, NOT the merged live file.
# This preserves Ruflo's unmodified output as the merge base for the next upgrade.
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

Recompute SHA-256 hashes. For `settings.json` and `CLAUDE.md`, hash the snapshot files (staging-sourced), not the live merged files:
```bash
shasum -a 256 <file> | awk '{print $1}'
```

Overwrite `manifest.json` with the new version, timestamp, and all file entries:
```json
{
  "rufloVersion": "<new-version>",
  "timestamp": "<ISO-8601-UTC>",
  "initCommand": "npx ruflo@latest init upgrade",
  "files": {
    ".claude/settings.json": { "hash": "<sha256>", "ownership": "shared",
      "_note": "hash of pure Ruflo staging output, not the merged file" },
    "CLAUDE.md":             { "hash": "<sha256>", "ownership": "section-managed",
      "_note": "hash of Ruflo sentinel block only, not the full merged file" },
    ".claude/agents/researcher.md": { "hash": "<sha256>", "ownership": "whole-file" },
    ".claude/agents/security.md":   { "hash": "<sha256>", "ownership": "whole-file" },
    ".claude/helpers/hook-handler.cjs": { "hash": "<sha256>", "ownership": "whole-file" }
  }
}
```

Include every managed file in the manifest — agents the user kept unchanged, new ones, helpers, commands. The snapshot must reflect the full current state.

The staging dir cleanup is handled by the `trap` set in Step 3.

---

## Step 8 — `.gitignore` check

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

## Step 9 — Final confirmation

```
✅ Upgrade complete. Ruflo <old-version> → <new-version>

Auto-applied (Ruflo updates, no user edits):
  .claude/settings.json          — 3 hooks updated, command paths fixed
  .claude/agents/researcher.md   — updated by Ruflo
  .claude/agents/security.md     — new agent added
  .claude/helpers/hook-handler.cjs — helper updated

Kept as-is (your edits, Ruflo unchanged):
  .claude/agents/coder.md        — your customisations preserved

Resolved with your input:
  CLAUDE.md                      — Ruflo section updated, your content untouched

Skipped (unchanged):
  .claude/commands/sparc.md

Snapshot updated to ruflo <new-version>
Pre-upgrade files at: .claude/backup-YYYYMMDD-HHMMSS/
Restore with:
  cp .claude/backup-YYYYMMDD-HHMMSS/settings.json .claude/settings.json
  cp .claude/backup-YYYYMMDD-HHMMSS/CLAUDE.md CLAUDE.md

Validation warnings:
  (none)
```

If fallback to `init --wizard` was used in Step 3, note it here.

---

## Edge cases

**Staging fails**: Stop immediately. Live project is untouched. Report the full error output.

**`init upgrade` subcommand missing**: Fall back silently to `init --wizard` in staging. Note the fallback in the final confirmation.

**User wants to skip a specific file**: Respect it. Mark it "skipped by user" in the final confirmation. Note it may be out of date.

**User wants to revert everything**: Print exact restore commands for each file from the backup directory, plus the snapshot:
```bash
cp .claude/backup-YYYYMMDD-HHMMSS/settings.json .claude/settings.json
cp .claude/backup-YYYYMMDD-HHMMSS/CLAUDE.md CLAUDE.md
cp -r .claude/backup-YYYYMMDD-HHMMSS/agents/. .claude/agents/
cp -r .claude/backup-YYYYMMDD-HHMMSS/helpers/. .claude/helpers/
cp -r .claude/backup-YYYYMMDD-HHMMSS/ruflo-snapshot/. .claude/ruflo-snapshot/
```

**`CLAUDE.md` sentinel markers missing** (project set up before this skill): Treat the entire file as user content. Do not modify it. Append a new marked Ruflo block at the bottom. Warn:
> "CLAUDE.md has no ruflo sentinel markers — treating the entire file as user content. On future upgrades, only the newly added marked block will be updated."

**Duplicate `<!-- ruflo:start -->` markers**: Fall back to conflict mode for `CLAUDE.md`. Show both occurrences and ask the user to resolve them manually before you can proceed.

**Conservative mode throughout**: When `CONSERVATIVE_MODE=1` (no snapshot), every file with any difference between OURS and THEIRS is surfaced as a potential conflict. Present the full diff and ask for each one. Be explicit about the reason: "I have no baseline to compare against, so I'm asking about this file to be safe."
