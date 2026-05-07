# intern-os × Claude Code — Adaptation Report

## What intern-os is

intern-os is Frutero's workstream framework: a file-based coordination layer for human+AI collaboration. Each project is a directory with `CLAUDE.md`, `AGENTS.md`, `TICK.md`, a `workstreams/` folder, and optionally a `references/` folder. Each workstream gets its own subdirectory with six standard files.

The framework was designed to be AI-agnostic, but the Claude Code adapter (`CLAUDE.md`) is what makes it work with this specific tool.

---

## What worked

**CLAUDE.md as the Claude Code adapter.** Placing `CLAUDE.md` at the project root is all that's needed for Claude Code to auto-load it at session start. The file tells Claude to: resolve the workstream by `thread_id`, read `AGENTS.md` + `BRIEF.md` + `STATUS.md` in that order, claim/release tasks via tick-md, and update `STATUS.md` before ending any session. This worked exactly as intended.

**CLAUDE.md ≈ AGENTS.md.** In Claude Code, `CLAUDE.md` is the project-level system prompt for the agent. In intern-os, `AGENTS.md` holds project context (stack, conventions, key people). These serve different functions but are both needed. Keep both.

**Workstream file structure.** The six-file layout (`BRIEF`, `STATUS`, `MEMORY`, `DECISIONS`, `STAKEHOLDERS`, `RESOURCES`) plus `docs/` maps naturally onto Claude Code sessions. `STATUS.md` (≤10 lines, mandatory update per session) is the most important — it's the only file Claude reads unconditionally, so it functions as the session handoff artifact.

**Claude auto-memory as a complement.** Claude Code has its own persistent memory at `~/.claude/projects/<path-hash>/memory/`. The right division: intern-os files hold durable project context; Claude auto-memory holds behavioral priors and critical gotchas that are harder to express in file structure (e.g., known bugs, collaboration preferences).

**`docs/journal.md` as the execution log.** Works well as an append-only log with inline tags (✓ ✗ 💡 ⚠️). Keeps session history without cluttering `MEMORY.md`.

---

## What didn't work / friction points

**tick-md flag inconsistency (v1.2.4).** The CLI has three different flag names across three commands:

| Command | Flag |
|---------|------|
| `tick agent register` | `--roles` (plural) |
| `tick add` | `--tags` (plural) |
| `tick list` | `--tag` (singular) |

intern-os docs and the CLAUDE.md adapter use singular throughout — this breaks `register` and `add` silently. Memorize the actual flags until an upstream fix lands.

**Cascading tool failures.** When a Bash command fails (e.g., wrong tick-md flag), all sibling Write/Edit calls in the same response get cancelled. Fix: never batch a Bash verification with bulk file writes in the same response turn.

**Framework path confusion (resolved after three correction cycles).** The initial setup had three wrong assumptions:

1. `WORKSTREAMS.md` belongs in the intern-os repo → it belongs in each project
2. intern-os is a "parent workspace" with nested projects → it's a sibling in `~/projects/`
3. `CLAUDE.md` has a different role than `AGENTS.md` → they're functionally equivalent; both are needed

These required explicit correction each time. The model inferred structure from the intern-os repo layout rather than from instruction. Save time by stating these three points explicitly when setting up a new device.

**Project naming.** Get the project name right at setup time. NemoClaw is a workstream, not the project — the project is `nebius-devrel`.

---

## Quick setup guide

**Prerequisites:** Node.js 22+, `tick-md` installed globally (`npm i -g tick-md`), `gh` CLI authenticated.

```bash
# 1. Clone intern-os (reference only — don't work inside it)
cd ~/projects
git clone https://github.com/fruteroclub/intern-os

# 2. Create the project directory
mkdir nebius-devrel && cd nebius-devrel

# 3. Copy the Claude Code adapter files from intern-os
cp ../intern-os/CLAUDE.md .
cp ../intern-os/AGENTS.md .        # then edit for this project's stack
cp ../intern-os/TICK.md .          # or create fresh

# 4. Initialize tick
tick init --force

# 5. Register agents (note the plural flags)
tick agent register @mel --roles human
tick agent register @claude-code --roles bot,engineer

# 6. Scaffold the workstream
mkdir -p workstreams/<workstream-name>/docs

# 7. Create the six workstream files
# BRIEF.md       — thread_id is mandatory (synthetic form: local:cli/YYYY-MM-DD-<slug>)
# STATUS.md      — current phase, next, blockers (≤10 lines)
# MEMORY.md      — durable context across sessions
# DECISIONS.md   — key decisions with date + rationale
# STAKEHOLDERS.md — relevant people and roles
# RESOURCES.md   — artifact registry

# 8. Add the first task (note the plural flag)
tick add "Phase 1: ..." --tags <workstream-name> --priority high
```

**Claude auto-memory setup** (optional but recommended for multi-session work):

```bash
# The path hash is derived from the absolute project path
mkdir -p ~/.claude/projects/-home-<user>-projects-<project>/memory/

# Create at minimum:
# MEMORY.md          — index file (one line per memory entry)
# project_pointer.md — tells Claude to follow CLAUDE.md → workstream files
# Any gotcha files   — critical bugs or constraints to re-surface every session
```

**To start a new session:**

```bash
cd ~/projects/nebius-devrel
claude  # CLAUDE.md is auto-loaded
```

Claude will read `WORKSTREAMS.md`, resolve the active workstream by `thread_id`, read `BRIEF.md` + `STATUS.md`, and be ready to work.

---

## One thing to carry forward

Always update `STATUS.md` before closing a session. It's the only file Claude reads unconditionally at session start — if it's stale, the handoff fails.
