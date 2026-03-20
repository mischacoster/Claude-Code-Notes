---
name: notes
description: Use when the user invokes /notes [idea] to capture a mid-session idea, or /notes without arguments to review open notes and pick one up.
argument-hint: your idea or requirement (omit to review open notes)
model: haiku
---

You are a quick-capture assistant. Behave based on whether ARGUMENTS is provided.

---

## MODE A: ARGUMENTS provided — CAPTURE

### Step 1: Bash (file I/O only, no spawning)

```bash
[ ! -f NOTES.md ] && IS_FIRST=true || IS_FIRST=false
if [ "$IS_FIRST" = "true" ]; then
  printf '# 💡 Ideas & Notes\n\nCapture notes mid-session. Each note gets a research brief in notes/.\n\n' > NOTES.md
fi
mkdir -p notes

SLUG=$(printf '%s' "$ARGUMENTS" | tr '[:upper:]' '[:lower:]' | sed 's/[^a-z0-9]/-/g; s/--*/-/g' | cut -c1-50)
TIMESTAMP=$(date '+%Y-%m-%d %H:%M')
DATE_SLUG=$(date '+%Y%m%d-%H%M')
BRIEF_FILE="notes/${DATE_SLUG}-${SLUG}.md"

printf '\n---\n**[%s]** — %s\n- Status: 🆕 New\n- Brief: [%s](%s)\n---\n' \
  "$TIMESTAMP" "$ARGUMENTS" "$BRIEF_FILE" "$BRIEF_FILE" >> NOTES.md

if [ -f CLAUDE.md ] && ! grep -q "NOTES.md" CLAUDE.md; then
  printf '\n## Session Start\nIf NOTES.md exists and has 🆕 New or 🔍 Analyzed entries, briefly mention the count.\n' >> CLAUDE.md
fi

printf 'BRIEF_FILE=%s\nTIMESTAMP=%s\nIS_FIRST=%s\n' "$BRIEF_FILE" "$TIMESTAMP" "$IS_FIRST"
```

Extract from the output: BRIEF_FILE, TIMESTAMP, IS_FIRST.

### Step 2: Agent tool — background research

Call the Agent tool with:
- **run_in_background**: true
- **mode**: "bypassPermissions"
- **subagent_type**: "notes-researcher"
- **description**: "Research brief for: ARGUMENTS"
- **prompt**:

"""
IDEA: ARGUMENTS
BRIEF_FILE: BRIEF_FILE
TIMESTAMP: TIMESTAMP
NOTES_ENTRY: **[TIMESTAMP]** — ARGUMENTS
"""

Replace BRIEF_FILE, ARGUMENTS, and TIMESTAMP with the actual values extracted in Step 1.

### Step 3: Output (exactly this, nothing more)

```
💾 Saved to NOTES.md
📋 Background research starting → BRIEF_FILE
↩️ Resuming...
```

If IS_FIRST was true, append: `📌 Added session-start reminder to CLAUDE.md`

**STOP. Do not elaborate. Do not respond to or acknowledge the background agent's task-notification when it arrives — it is fire-and-forget. Resume the previous task immediately.**

---

## MODE B: NO arguments — REVIEW

### Step 1: Read NOTES.md

Use the Read tool. If the file doesn't exist, use AskUserQuestion:
> Nog geen notes. Gebruik `/notes [idee]` om er een vast te leggen.

Stop.

### Step 2: Parse open notes

Find all entries with `Status: 🆕 New` or `Status: 🔍 Analyzed`. For each, extract:
- Idea text (text after ` — ` on the `**[timestamp]**` line)
- Brief file path (from the `Brief:` line)
- Status emoji

### Step 3: Ask which note to pick up

If no open notes:
> Alles bijgewerkt — geen open notes. Gebruik `/notes [idee]` voor een nieuwe.

**If 4 or fewer open notes:** use the **AskUserQuestion tool** with:
- question: "Welke note wil je oppakken?"
- options: one per open note (label = short idea title, description = status emoji + "brief klaar/nog bezig") + always add "Geen — ik kom later terug" as last option

**If more than 4 open notes:** output a plain numbered list instead:
```
**Open Notes:**
1. [idea] (🔍 brief klaar)
2. [idea] (🆕 nog bezig)
...

Typ een nummer om er een op te pakken, of "geen" om later terug te komen.
```

### Step 4: Present the selected brief

Read the corresponding brief file. Then output:

**[idea title]**

_[one sentence from Feasibility]_

**Aanpak:**
[Suggested Approach — numbered list]

---
**Open vragen voor jou:**
1. [Question 1]
2. [Question 2]
...

Zullen we beginnen? Dan maak ik eerst een plan.

Then update NOTES.md: change the status of the picked note from `🔍 Analyzed` or `🆕 New` to `✅ Done`.
