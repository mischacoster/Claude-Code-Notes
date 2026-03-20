# /notes

**Capture ideas mid-session without breaking your flow.**

---

You're deep in a debugging session. Suddenly you realize the authentication flow could use a retry mechanism. It's not related to what you're doing right now, and you don't want to lose your train of thought — but you also don't want to forget it.

`/notes` gives you a place to put it. One command captures the idea, a background agent researches it in your codebase, and when you're ready, you pick it up with a structured brief and suggested approach.

> *"Claude Code has memory for preferences, todos for tracking current work, and plans for structuring implementation — but nothing for the ideas that pop up in between. The ones that aren't about what you're doing right now, but what you might want to do next. You could mention them in chat, but they'd dissolve into context history. You could try to remember them yourself, but that's exactly the kind of cognitive overhead an AI assistant should eliminate. `/notes` fills that gap: capture now, research in the background, pick up whenever you're ready."*
>
> — Claude, when asked why this needed to exist

## What it does

Two modes, one command:

**Capture** — `/notes [idea]`

Saves the idea to `NOTES.md` and dispatches a background agent to research it in your codebase. You see a quick confirmation and immediately continue working:

```
💾 Saved to NOTES.md
📋 Background research starting → notes/20260320-1425-auth-retry-mechanism.md
↩️ Resuming...
```

**Review** — `/notes` (no arguments)

Shows your open notes and lets you pick one up. The research brief includes relevant files, architecture context, a suggested approach, and open questions to consider before starting.

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

You should see a confirmation message, a new `NOTES.md` file, and a research brief appearing in `notes/`.

## Usage

### Capturing an idea

```
/notes add dark mode support to the settings page
/notes the onboarding flow needs a progress indicator
/notes refactor the API client to use retry logic
```

Each captured note gets:
- An entry in `NOTES.md` with a timestamp
- A background research brief written to `notes/`

### Reviewing and picking up notes

```
/notes
```

This shows all open notes with their status. Select one to see the research brief, which includes:
- **Relevant files** found in your codebase
- **Architecture context** — how this idea fits into existing patterns
- **Suggested approach** — concrete steps to get started
- **Open questions** — things to consider before implementing

### Status flow

| Status | Meaning |
|--------|---------|
| New | Just captured, research brief in progress |
| Analyzed | Brief is ready, waiting to be picked up |
| Done | Picked up and worked on |

## How it works

The `/notes` skill is a two-part system:

1. **The skill** (`~/.claude/skills/notes/SKILL.md`) handles the slash command — it saves the note to `NOTES.md` and dispatches the research agent.

2. **The agent** (`~/.claude/agents/notes-researcher.md`) runs in the background using Haiku for speed. It scans your codebase with Glob, Grep, and Read, then writes a structured research brief. Everything happens async — you continue working while it runs.

The agent writes files using Bash (not the Write tool) due to how permissions work for background agents in Claude Code. This is intentional, not a workaround.

## Language

The skill follows the language of your input. Write your note in English, the brief comes back in English. Write it in Dutch, you get Dutch. Code identifiers and file paths always stay in their original form.

## Known limitations

- **Max 4 options in the review picker** — when you have more than 4 open notes, the skill falls back to a numbered text list instead of the interactive picker.
- **Restart required after first install** — custom agents need to be indexed on Claude Code startup.
- **Background agents can't use the Write tool** — file writing uses Bash heredocs. This is a Claude Code platform constraint.
- **The `model: haiku` frontmatter** may show a linter warning in your IDE, but works correctly at runtime.

## License

[MIT](LICENSE)
