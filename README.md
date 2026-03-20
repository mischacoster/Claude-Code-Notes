# /notes

**Capture ideas mid-session without breaking your flow.**

---

You're deep in a debugging session. Suddenly you realize the authentication flow could use a retry mechanism. It's not related to what you're doing right now, and you don't want to lose your train of thought — but you also don't want to forget it.

`/notes` gives you a place to put it. One command captures the idea, and when you're ready, you pick it up. Add the `-r` flag and a background agent researches it in your codebase while you keep working.

> *"Claude Code has memory for preferences, todos for tracking current work, and plans for structuring implementation — but nothing for the ideas that pop up in between. The ones that aren't about what you're doing right now, but what you might want to do next. You could mention them in chat, but they'd dissolve into context history. You could try to remember them yourself, but that's exactly the kind of cognitive overhead an AI assistant should eliminate. `/notes` fills that gap: capture now, research in the background, pick up whenever you're ready."*
>
> — Claude, when asked why this needed to exist

## What it does

Three modes, one command:

**Quick capture** — `/notes [idea]`

Saves the idea to `NOTES.md`. You see a confirmation and immediately continue working:

```
💾 Saved to NOTES.md
↩️ Resuming...
```

**Capture + research** — `/notes -r [idea]`

Same as above, but also dispatches a background Haiku agent that scans your codebase (and the web, when relevant). It writes a structured research brief while you keep working:

```
💾 Saved to NOTES.md
📋 Background research starting → notes/20260320-1425-auth-retry-mechanism.md
↩️ Resuming...
```

**Review** — `/notes` (no arguments)

Shows your open notes and lets you pick one up. If background research was done, the brief includes relevant files, architecture context, a suggested approach, and open questions to consider before starting.

## Installation

The skill consists of two files: a slash command and a background research agent.

### 1. Copy the files

```bash
# Create directories if they don't exist
mkdir -p ~/.claude/skills/notes
mkdir -p ~/.claude/agents

# Copy skill and agent
cp skills/notes/SKILL.md ~/.claude/skills/notes/SKILL.md
cp agents/notes-researcher.md ~/.claude/agents/notes-researcher.md
```

### 2. Restart Claude Code

The agent needs to be indexed on startup. A restart is required after first installation.

### 3. Verify

```
/notes test idea
```

You should see a confirmation message and a new `NOTES.md` file in your project root.

To test background research (requires bypass permissions):

```
/notes -r test idea with research
```

You should also see a research brief appearing in `notes/`.

## Usage

### Quick capture

```
/notes add dark mode support to the settings page
/notes the onboarding flow needs a progress indicator
/notes refactor the API client to use retry logic
```

Each captured note gets a timestamped entry in `NOTES.md`.

### Capture with background research

```
/notes -r add dark mode support to the settings page
/notes -r investigate why the webhook occasionally times out
```

Same as quick capture, plus a background agent scans your codebase and writes a structured research brief to `notes/`.

### Reviewing and picking up notes

```
/notes
```

This shows all open notes with their status. Select one to see details. For notes with a research brief, you get:
- **Relevant files** found in your codebase
- **Architecture context** — how this idea fits into existing patterns
- **Suggested approach** — concrete steps to get started
- **Open questions** — things to consider before implementing

For notes without a brief (quick capture), you can choose to research on the spot or start working directly.

### Status flow

| Status | Meaning |
|--------|---------|
| 📝 Captured | Saved, no background research |
| 🆕 New | Saved, research brief in progress |
| 🔍 Analyzed | Research brief is ready |
| ✅ Done | Picked up and worked on |

## How it works

The `/notes` skill is a two-part system:

1. **The skill** (`~/.claude/skills/notes/SKILL.md`) handles the slash command — it saves the note to `NOTES.md` and optionally dispatches the research agent.

2. **The agent** (`~/.claude/agents/notes-researcher.md`) runs in the background using Haiku for speed. It scans your codebase with Glob, Grep, and Read, searches the web when relevant, then writes a structured research brief. Everything happens async — you continue working while it runs.

The agent writes files using Bash heredocs (not the Write tool) due to how permissions work for background agents in Claude Code. This is a platform constraint, not a workaround.

## Language

The skill follows the language of your input. Write your note in English, the brief comes back in English. Write it in Dutch, you get Dutch. Code identifiers and file paths always stay in their original form.

## Permissions

The skill is designed to work across all Claude Code permission modes.

| Mode | `/notes [idea]` | `/notes -r [idea]` | `/notes` (review) |
|------|-----------------|--------------------|--------------------|
| Bypass permissions | No prompts | No prompts | No prompts |
| Ask before edits | Once per session | Not available | Prompts per action |
| Auto accept edits | Once per session | Not available | Prompts per action |

**Quick capture works for everyone.** The first time you use `/notes [idea]` in a session, Claude Code asks permission for the Write tool. Choose "Allow for this session" and you're set — no more prompts until you restart.

**Background research requires bypass permissions.** The research agent runs in the background and needs Bash access to write the brief file. Background agents in Claude Code don't inherit tool permissions from the parent session, so only bypass permissions grants this. Without it, the agent runs but can't save its output — wasting tokens with no result. That's why research is opt-in with the `-r` flag.

> **Why not make research the default?** Without bypass permissions, the research agent silently fails — it uses tokens but produces no output. We don't want anyone unknowingly burning through Haiku credits for a brief that never gets written.

## Known limitations

- **Max 4 options in the review picker** — when you have more than 4 open notes, the skill falls back to a numbered text list instead of the interactive picker. This is a Claude Code platform limit.
- **Restart required after first install** — custom agents need to be indexed on Claude Code startup. Skill changes take effect immediately.
- **Background agents can't use the Write tool** — file writing uses Bash heredocs. This is a Claude Code platform constraint, not a bug.
- **VS Code shows a busy indicator** while the background agent runs. This is cosmetic — you can continue working normally.

## License

[MIT](LICENSE)
