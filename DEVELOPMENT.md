# Development History

This document captures the iteration journey and design decisions behind `/notes`. Useful if you're building your own Claude Code skills/agents or want to understand why things work the way they do.

## The Problem

During Claude Code sessions, ideas and requirements pop up that aren't related to the current task. Without a capture system, they get lost or break your workflow. Claude Code's built-in tools don't cover this:

- **Memory** (`MEMORY.md`) — stores user preferences and project context, not task ideas
- **Todos** (`TodoWrite`) — tracks progress on the *current* task, resets between sessions
- **Plans** — structure implementation of a known goal, not a place for unformed ideas
- **Chat history** — ideas mentioned in passing dissolve into compressed context

## Iteration Journey

### Attempt 1: `nohup claude -p` (failed)

The original skill tried to spawn a background Claude CLI process from Bash using `nohup claude -p "..." &`. This doesn't work inside the Claude Code environment — the `claude` CLI isn't available as a subprocess in this context.

### Attempt 2: Agent tool + Write tool (failed)

Switched to using Claude Code's built-in Agent tool with `run_in_background: true`. The background agent could read the codebase but couldn't write files — the Write tool requires permissions that spawned agents don't inherit, even with `mode: "bypassPermissions"` on the Agent call.

### Attempt 3: Agent tool + Bash heredoc (works)

The breakthrough: instead of using the Write tool, the agent writes files via `cat > file.md << 'EOF'` in Bash. Bash file writing works where the Write tool doesn't — but only when the parent session has bypass permissions enabled.

### Attempt 4: Custom `notes-researcher` agent (works, final)

Created a dedicated agent in `~/.claude/agents/notes-researcher.md` with:
- `model: haiku` for speed (instead of inheriting the parent's Opus/Sonnet)
- Focused tool set: Glob, Grep, Read, Bash, WebSearch, WebFetch
- Focused prompt optimized for research briefs
- Requires a Claude Code restart after first installation to be recognized

### Attempt 5: Permission-aware dual mode (current)

Testing revealed that the background agent silently fails without bypass permissions — it runs, uses tokens, but can't write output. This "token leak" led to splitting the skill into two modes:

- **Default** (`/notes [idea]`): quick capture using Read/Write tools. Works with any permission mode. First use per session requires one approval ("Allow for this session").
- **Research** (`/notes -r [idea]`): capture + background research agent. Requires bypass permissions.

This ensures nobody unknowingly wastes tokens on a research agent that can't save its results.

## Key Technical Findings

### Permission model deep dive

Extensive testing across all Claude Code permission modes revealed:

| Permission mode | Skill's own Bash | Agent dispatch | Agent's internal Bash |
|---|---|---|---|
| Ask before edits | Yes/No prompt each time | Yes/No prompt each time | Silently blocked |
| Auto accept edits | Yes/No prompt each time | Yes/No prompt each time | Silently blocked |
| Bypass permissions | Automatic | Automatic | Works |

Key insights:
- `mode: "bypassPermissions"` on the Agent tool call does **not** propagate to the agent's own tool permissions
- The Write/Edit tools offer "Allow for this session" — a session-scoped approval that resets on VS Code restart
- Skills appear to have automatic edit rights (Edit/Write tool prompts don't appear), but Bash and Agent tool calls still prompt
- "Session" for permission purposes is tied to the VS Code extension runtime, not the chat conversation

### Background agents in VS Code

- `run_in_background: true` truly runs async — you CAN send new messages and use tools while it runs
- VS Code shows a busy indicator while the background agent works. This is cosmetic — you can continue working normally.
- Task notifications from completed agents trigger a new "busy" cycle — mitigated by adding "fire-and-forget" instruction to the skill

### Custom agents vs general-purpose

- Custom agents in `~/.claude/agents/` ARE recognized by the Agent tool's `subagent_type` parameter
- They require a Claude Code restart after creation to be indexed. Skill changes take effect immediately.
- They can specify their own model (haiku/sonnet/opus) independent of the parent session

### AskUserQuestion tool

- Limited to max 4 options per question (schema-enforced `maxItems: 4`)
- The skill falls back to plain text numbered lists when >4 notes exist

### Haiku language behavior

- Haiku defaults to English even when input is in another language
- Requires explicit, emphatic instructions ("CRITICAL", "MUST") to maintain language matching

## Architecture Decisions

### Why a separate Haiku agent instead of inline research?

- **Speed**: Haiku is fast and cheap for research tasks
- **Independence**: doesn't consume the parent's context window
- **Focused**: has only the tools it needs

### Why Bash heredoc instead of Write tool?

- Write tool requires permissions that background agents don't reliably get
- Bash file writing (`cat > file << 'EOF'`) works with `mode: "bypassPermissions"`
- Simpler — no tool permission negotiation needed

### Why fire-and-forget?

- Task notifications trigger the VS Code "busy" indicator
- The agent writes files and updates NOTES.md autonomously
- No parent intervention needed — the user sees the file appear in their explorer

### Why Read/Write for default mode, Bash for research mode?

- Read/Write tools offer "Allow for this session" — one prompt per session
- Bash prompts with Yes/No every time (no session-wide option)
- Default mode (quick capture) should have minimal friction for all users
- Research mode already requires bypass permissions, so Bash is fine there

### Why research is opt-in (`-r`), not the default?

- Without bypass permissions, the research agent runs but can't write output
- This wastes Haiku tokens with no visible result (a "token leak")
- Making research opt-in ensures users consciously choose it when their setup supports it

### Why AskUserQuestion for ≤4, text list for >4?

- AskUserQuestion has a hard limit of 4 options
- Text list scales to any number of notes
- Notes should be archived (Done) after pickup to keep the list short
