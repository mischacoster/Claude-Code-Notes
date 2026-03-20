# claude-notes-skill

A `/notes` skill + background research agent for Claude Code. Capture ideas mid-session without breaking your flow.

## What This Is

A two-part system for Claude Code:
1. **Skill** (`/notes`) — slash command to capture or review ideas
2. **Agent** (`notes-researcher`) — background Haiku agent that researches the idea in your codebase

## Development History & Design Decisions

### The Problem
During Claude Code sessions, ideas and requirements pop up that aren't related to the current task. Without a capture system, they get lost or break your workflow.

### Iteration Journey

**Attempt 1: `nohup claude -p` (failed)**
The original skill tried to spawn a background Claude CLI process from Bash using `nohup claude -p "..." &`. This doesn't work inside the Claude Code environment — the `claude` CLI isn't available as a subprocess in this context.

**Attempt 2: Agent tool with `run_in_background: true` + Write tool (failed)**
Switched to using Claude Code's built-in Agent tool. The background agent could read the codebase but couldn't write files — the Write tool requires permissions that spawned agents don't inherit, even with `mode: "bypassPermissions"`.

**Attempt 3: Agent tool + Bash heredoc for writing (works!)**
The breakthrough: instead of using the Write tool, the agent writes files via `cat > file.md << 'EOF'` in Bash. Bash file writing works where the Write tool doesn't.

**Attempt 4: Custom agent `notes-researcher` (works!)**
Created a dedicated agent in `~/.claude/agents/notes-researcher.md` with:
- `model: haiku` for speed (instead of inheriting the parent's Opus/Sonnet)
- Minimal tool set: only Glob, Grep, Read, Bash
- Focused prompt optimized for research briefs
- Requires a Claude Code restart after first installation to be recognized

### Key Technical Findings

**Background agents in VS Code:**
- `run_in_background: true` truly runs async — you CAN send new messages while it runs
- The "busy" indicator in VS Code stays on while you (the parent) are processing, NOT because of the background agent
- Task notifications from completed agents trigger a new "busy" cycle — mitigated by adding "fire-and-forget" instruction to the skill

**Custom agents vs general-purpose:**
- Custom agents in `~/.claude/agents/` ARE recognized by the Agent tool's `subagent_type` parameter
- They require a Claude Code restart after creation to be indexed
- They can specify their own model (haiku/sonnet/opus) independent of the parent session

**AskUserQuestion tool:**
- Limited to max 4 options per question (schema-enforced `maxItems: 4`)
- The skill falls back to plain text numbered lists when >4 notes exist
- The tool had bugs previously (auto-submitting without user input) but these appear fixed

**Haiku language behavior:**
- Haiku defaults to English even when input is Dutch
- Requires explicit, emphatic instructions ("CRITICAL", "MUST", "do NOT default to English") to maintain language matching

### Architecture Decisions

**Why a separate agent instead of inline research?**
- Speed: Haiku is fast and cheap for research tasks
- Independence: doesn't consume the parent's context window
- Focused: has only the tools it needs (no Write, no MCP, no web)

**Why Bash heredoc instead of Write tool?**
- Write tool requires permissions that background agents don't reliably get
- Bash file writing (`cat > file << 'EOF'`) works with `mode: "bypassPermissions"`
- Simpler — no tool permission negotiation needed

**Why fire-and-forget?**
- Task notifications trigger the VS Code "busy" indicator
- The agent writes files and updates NOTES.md autonomously
- No parent intervention needed — the user sees the file appear in their explorer

**Why AskUserQuestion for ≤4, text list for >4?**
- AskUserQuestion has a hard limit of 4 options
- Text list scales to any number of notes
- Notes should be archived (✅ Done) after pickup to keep the list short

## Installation

### Files to copy:

```
~/.claude/skills/notes/SKILL.md    → The /notes slash command
~/.claude/agents/notes-researcher.md → The background research agent
```

### After installation:
1. Restart Claude Code (the agent needs to be indexed)
2. Test with `/notes test idea` — should save to NOTES.md and create a brief in `notes/`
3. Test with `/notes` (no args) — should show open notes to pick up

## Usage

```
/notes [idea]     → Capture idea + start background research
/notes            → Review open notes, pick one up
```

### Status flow:
- 🆕 New → just captured, research brief in progress
- 🔍 Analyzed → brief is ready, waiting to be picked up
- ✅ Done → picked up and worked on

## Known Limitations

- The `model: haiku` in SKILL.md frontmatter may trigger an IDE linter warning ("not supported") but works at runtime
- Background agents can't use the Write tool — must use Bash heredoc
- AskUserQuestion maxes out at 4 options
- Haiku needs strong language instructions to not default to English
- First-time agent installation requires Claude Code restart
