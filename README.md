# ruflo-safe

Two Claude Code skills that make [Ruflo](https://github.com/ruvnet/ruflo) (also published as `claude-flow`) safe to install and upgrade in projects that already have existing Claude Code configuration.

---

## The problem

Ruflo is an agent orchestration framework that runs on top of Claude Code. To set itself up, it writes directly into several Claude Code configuration files:

- `.claude/settings.json` — hooks, permissions, tool allowances
- `.claude/settings.local.json` — local permission overrides
- `CLAUDE.md` — project instructions
- `.claude/agents/*.md` — agent definitions
- `.claude/commands/*` — slash commands
- `.claude/helpers/` — runtime helper scripts called by hooks
- `.mcp.json` — MCP server registrations
- `claude-flow.config.json` — Ruflo configuration

**Ruflo's init is destructive by default.** Running `npx ruflo@latest init --wizard` in a project that already has Claude Code configuration will silently overwrite whatever was there. There is no dry-run, no merge, no confirmation step.

This has caused real data loss. Known failure modes include:

- Existing MCP server registrations erased
- Custom hooks overwritten with Ruflo's hooks
- Ruflo-generated `settings.json` referencing helper files that Ruflo never actually wrote (causing silent hook failures at runtime)
- Invalid hook event names written into `settings.json`

Ruflo's upgrade command (`npx ruflo@v3alpha init upgrade`) documents that it "preserves your data without overwriting your customisations" — but based on its behaviour it implements **skip-if-exists**, not merge. If a file already exists it leaves it alone; if it doesn't, it creates it. There is no 3-way merge, no conflict detection, no diff.

---

## What these skills do

Two Claude Code skills wrap Ruflo's init and upgrade commands with a safe lifecycle:

1. **Stage** — run Ruflo in a temp directory, never in the live project
2. **Classify** — compare what Ruflo wants to write against what you currently have (and, on upgrade, against what the previous Ruflo wrote)
3. **Propose** — show a merged plan with every decision made explicit
4. **Approve** — wait for explicit user confirmation before touching any live file
5. **Apply atomically** — write to temp paths, validate, then rename; never leave the project in a partial state
6. **Snapshot** — record exactly what Ruflo wrote, so future upgrades can do accurate 3-way merging

| Skill | Use when |
|---|---|
| `ruflo-init` | Adding Ruflo to a project for the first time |
| `ruflo-upgrade` | Updating an existing Ruflo installation |

---

## Core concepts

### The staging directory

Every Ruflo command runs in a `mktemp -d` staging directory — never in the live project. The live project is not touched until the user has seen the full plan and approved it. A `trap` ensures the staging directory is cleaned up on exit regardless of success or failure.

### The snapshot

After every successful init or upgrade, the skills save a copy of everything Ruflo wrote to `.claude/ruflo-snapshot/`. This is not a backup — it is the **merge base**.

```
.claude/ruflo-snapshot/
├── manifest.json       ← version, timestamp, per-file hashes and ownership modes
├── settings.json
├── CLAUDE.md
├── agents/
├── commands/
└── helpers/
```

On the next upgrade, the snapshot is the reference that lets the skill distinguish:
- "Ruflo changed this" (BASE→THEIRS differs, BASE→OURS is the same) → auto-apply
- "You changed this" (BASE→OURS differs, BASE→THEIRS is the same) → keep yours
- "Both changed this differently" → genuine conflict, always ask

Without the snapshot, ruflo-upgrade falls back to conservative mode and asks about every changed file.

**What the snapshot stores matters.** For `settings.json` (shared) the snapshot is a copy of Ruflo's raw staging output — not the merged file that was written to the project. For `CLAUDE.md` (section-managed) the snapshot stores only the Ruflo sentinel block, not the full file. This distinction is critical: when the next upgrade diffs BASE vs OURS, it is comparing Ruflo's pure output against the current state, not the user's merged result against itself.

### Ownership modes

Every managed file has an ownership mode that determines its merge strategy:

| Mode | Files | Strategy |
|---|---|---|
| `shared` | `settings.json` | Key-level JSON merge |
| `section-managed` | `CLAUDE.md` | Replace only the sentinel block |
| `whole-file` | agents, commands, helpers | Full file replace |

### `settings.json` merge

`settings.json` is merged at the JSON key level, never by line.

**Hooks** are merged semantically. The identity of a hook is `(event, normalizedCommand)` — normalised by stripping path prefix variants and collapsing whitespace. Hooks with the same identity are deduplicated. Hooks unique to the user are kept. Hooks unique to Ruflo are appended. Hooks with the same identity but different bodies are surfaced as conflicts.

**Permissions and allowedTools** are unioned. A permission the user had is never removed.

**All other keys** prefer the user's value. If Ruflo's value differs, the skill flags it but does not block.

### `CLAUDE.md` sentinel markers

Ruflo's content in `CLAUDE.md` is wrapped in hard markers:

```markdown
<!-- ruflo:start version=3.5.15 -->
[all ruflo-managed content]
<!-- ruflo:end -->
```

Content outside the markers is user-owned and is never modified. On upgrade, only the content between the markers is replaced. If markers are absent (project predates this skill), the entire file is treated as user content and the Ruflo block is appended.

**Conflict classification for `CLAUDE.md` is scoped to the sentinel block.** When ruflo-upgrade runs the 3-way diff, it extracts the block between the markers from both OURS and the snapshot, and diffs only those. User edits outside the markers are invisible to conflict detection — they can never trigger a conflict, and they are never touched during apply.

### Helper files

Ruflo installs helper scripts (`.claude/helpers/*.cjs`, `.sh`, etc.) that hook commands in `settings.json` call at runtime. If an upgrade updates hook command paths but not the helper files they reference, the project silently breaks.

The skills treat helper files as first-class managed artifacts — they are included in backup, staging diff, snapshot, and atomic apply on equal footing with `settings.json`.

### Atomic apply

The apply sequence never writes directly to a live path:

1. Write each resolved file to `<path>.ruflo-tmp`
2. Validate (`settings.json` gets a JSON parse check)
3. If all validations pass: rename each `.ruflo-tmp` to its final path
4. Copy new files (agents, helpers) from staging
5. Run soft validation and report warnings

If validation fails, all `.ruflo-tmp` files are deleted and the live project is untouched.

### Soft validation

After every merge, the skills run a set of non-blocking checks and surface the results as warnings in the proposal:

- JSON syntax valid
- Hook event names recognised by the skill's validator: `PreToolUse`, `PostToolUse`, `PostToolUseFailure`, `Stop`, `SubagentStop`, `SubagentStart`, `Notification`, `UserPromptSubmit`, `SessionStart`, `SessionEnd`, `PreCompact`, `PermissionRequest` — anything not on this list is flagged as "unrecognised by this skill's validator" (not necessarily invalid; Claude Code may support events the skill doesn't yet know about)
- All file paths referenced in hook commands exist on disk
- No duplicate hooks after normalisation
- Permission patterns use `:*` suffix for prefix matching

Warnings are never blocking — they inform the user but do not prevent apply.

### Two-tier approval

1. **Global plan** — a full summary of every file and action. The user approves or requests changes before anything is written.
2. **Per-conflict decisions** — for genuine conflicts only, shown one at a time with both versions and a proposed resolution.

No files are written until the user says "apply" (or equivalent).

---

## Install

### Global (available across all your projects)

```bash
mkdir -p ~/.claude/skills/ruflo-init ~/.claude/skills/ruflo-upgrade

curl -o ~/.claude/skills/ruflo-init/SKILL.md \
  https://raw.githubusercontent.com/jeffersonqiu/ruflo-safe/main/ruflo-init/SKILL.md

curl -o ~/.claude/skills/ruflo-upgrade/SKILL.md \
  https://raw.githubusercontent.com/jeffersonqiu/ruflo-safe/main/ruflo-upgrade/SKILL.md
```

### Project-level (committed to a specific repo)

```bash
mkdir -p .claude/skills/ruflo-init .claude/skills/ruflo-upgrade

curl -o .claude/skills/ruflo-init/SKILL.md \
  https://raw.githubusercontent.com/jeffersonqiu/ruflo-safe/main/ruflo-init/SKILL.md

curl -o .claude/skills/ruflo-upgrade/SKILL.md \
  https://raw.githubusercontent.com/jeffersonqiu/ruflo-safe/main/ruflo-upgrade/SKILL.md
```

---

## Usage

Once installed, invoke the skills from inside Claude Code:

```
/ruflo-init      ← first-time Ruflo setup
/ruflo-upgrade   ← upgrade an existing Ruflo installation
```

Claude Code will walk through each step interactively, showing you what it plans to do before writing anything.

---

## What gets created

**During init or upgrade:**

```
.claude/backup-YYYYMMDD-HHMMSS/   ← full copy of everything before Ruflo ran
.claude/ruflo-snapshot/            ← merge base for future upgrades (not a backup)
```

**`.gitignore` recommendation**: Both skills offer to add these paths to `.gitignore`. The snapshot is machine-local state and should generally not be committed alongside your project source.

---

## Relationship between the two skills

```
ruflo-init
  └── produces .claude/ruflo-snapshot/   ← shared artifact

ruflo-upgrade
  └── consumes .claude/ruflo-snapshot/   ← merge base
  └── updates  .claude/ruflo-snapshot/   ← after apply
```

The snapshot is the contract between the two skills. If it exists, upgrade can operate mostly automatically — only surfacing genuine conflicts. If it doesn't exist, upgrade degrades gracefully to conservative mode.

---

## What these skills deliberately do not do

- No ruflo-rollback skill — the backup directory and printed restore commands are sufficient
- No hard schema validation that blocks apply — warnings only
- No reserved namespace for agent filenames
- No network calls beyond running `npx ruflo@latest`
