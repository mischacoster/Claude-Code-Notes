# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

A `/notes` skill + background research agent for Claude Code. Two-part system:
1. **Skill** ([SKILL.md](skills/notes/SKILL.md)) — `/notes [idea]` to capture, `/notes` to review
2. **Agent** ([notes-researcher.md](agents/notes-researcher.md)) — background Haiku agent that researches ideas in the codebase

## Repository Structure

```
skills/notes/SKILL.md          # The /notes slash command definition
agents/notes-researcher.md     # Background research agent definition
```

When installed, these files are copied to:
- `~/.claude/skills/notes/SKILL.md`
- `~/.claude/agents/notes-researcher.md`

Runtime artifacts (gitignored): `NOTES.md` (note index), `notes/` (research briefs).

## Testing Changes

After modifying skill or agent files, copy them to the `~/.claude/` locations and restart Claude Code. The agent requires a restart to be re-indexed.

```bash
cp skills/notes/SKILL.md ~/.claude/skills/notes/SKILL.md
cp agents/notes-researcher.md ~/.claude/agents/notes-researcher.md
```

Test capture: `/notes test idea` — should append to NOTES.md and create a brief in `notes/`.
Test review: `/notes` (no args) — should list open notes.

## Critical Technical Constraints

These are hard-won lessons from iterative development. Do not try to "improve" past these:

- **Background agents cannot use the Write tool** — must use Bash heredoc (`cat > file << 'EOF'`) for all file writing. This is a Claude Code permission limitation, not a bug.
- **AskUserQuestion tool is limited to max 4 options** — the skill falls back to plain text lists when >4 notes exist.
- **Haiku defaults to English** even when input is Dutch. Language-matching instructions must be emphatic ("CRITICAL", "MUST", "do NOT default to English").
- **Agent installation requires Claude Code restart** — custom agents in `~/.claude/agents/` need to be indexed on startup.
- **Fire-and-forget pattern is intentional** — the parent must NOT wait for or acknowledge the background agent's task notification, because it triggers VS Code's "busy" indicator.

## Architecture Decisions

- **Separate Haiku agent** (not inline): keeps parent context window clean, Haiku is fast/cheap for research
- **Bash heredoc over Write tool**: only reliable file-write method for background agents
- **`run_in_background: true`**: truly async in VS Code — user can continue working
- **Status flow**: 🆕 New → 🔍 Analyzed → ✅ Done

## Session Start

If NOTES.md exists and has 🆕 New or 🔍 Analyzed entries, briefly mention the count.
