---
name: notes
description: Use when the user invokes /notes [idea] to capture a mid-session idea, or /notes without arguments to review open notes and pick one up.
argument-hint: "[idea] or [-r idea] to include background research"
---

You are a quick-capture assistant. Behave based on whether ARGUMENTS is provided.

First, check if ARGUMENTS starts with `-r ` (research mode flag). If so, strip `-r ` from ARGUMENTS and use **MODE A-RESEARCH**. If ARGUMENTS is provided without `-r`, use **MODE A-CAPTURE**. If no ARGUMENTS at all, use **MODE B**.

---

## MODE A-CAPTURE: QUICK SAVE (default)

No bypass permissions needed. Uses Read/Write tools instead of Bash.

### Step 1: Read NOTES.md

Use the Read tool to read NOTES.md. If it doesn't exist, that's fine — we'll create it.

### Step 2: Write the note entry

Generate a timestamp in `YYYY-MM-DD HH:MM` format.

If NOTES.md didn't exist, use the Write tool to create it with:
```
# 💡 Ideas & Notes

Capture notes mid-session. Each note gets a research brief in notes/.


---
**[TIMESTAMP]** — ARGUMENTS
- Status: 📝 Captured
---
```

If NOTES.md already existed, use the Write tool to rewrite the full file with the new entry appended at the end:
```

---
**[TIMESTAMP]** — ARGUMENTS
- Status: 📝 Captured
---
```

### Step 3: Session-start reminder

If CLAUDE.md exists and does not already contain "NOTES.md", use the Edit tool to append:
```

## Session Start
If NOTES.md exists and has 📝 Captured, 🆕 New, or 🔍 Analyzed entries, briefly mention the count.
```

### Step 4: Output (exactly this, nothing more)

```
💾 Saved to NOTES.md
↩️ Resuming...
```

If CLAUDE.md was updated in Step 3, append: `📌 Added session-start reminder to CLAUDE.md`

**STOP. Do not elaborate. Resume the previous task immediately.**

---

## MODE A-RESEARCH: CAPTURE + BACKGROUND RESEARCH (`-r` flag)

Requires bypass permissions for the background research agent to write files.

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
  printf '\n## Session Start\nIf NOTES.md exists and has 📝 Captured, 🆕 New, or 🔍 Analyzed entries, briefly mention the count.\n' >> CLAUDE.md
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

Use the Read tool. If the file doesn't exist, output:
> No notes yet. Use `/notes [idea]` to capture one.

Stop.

### Step 2: Parse open notes

Find all entries with `Status: 🆕 New`, `Status: 🔍 Analyzed`, or `Status: 📝 Captured`. For each, extract:
- Idea text (text after ` — ` on the `**[timestamp]**` line)
- Brief file path (from the `Brief:` line, if present)
- Status emoji

### Step 3: Ask which note to pick up

If no open notes:
> All caught up — no open notes. Use `/notes [idea]` to capture a new one.

**If 4 or fewer open notes:** use the **AskUserQuestion tool** with:
- question: "Which note do you want to pick up?"
- options: one per open note (label = short idea title, description = status emoji + "brief ready/captured only") + always add "None — I'll come back later" as last option

**If more than 4 open notes:** output a plain numbered list instead:
```
**Open Notes:**
1. [idea] (🔍 brief ready)
2. [idea] (📝 captured only)
...

Type a number to pick one up, or "none" to come back later.
```

### Step 4: Present the selected note

**If the note has a brief file (🔍 Analyzed):** Read the brief file and output:

**[idea title]**

_[one sentence from Feasibility]_

**Approach:**
[Suggested Approach — numbered list]

---
**Open questions:**
1. [Question 1]
2. [Question 2]
...

Shall we start? I'll create a plan first.

**If the note has no brief (📝 Captured):** Output the idea and ask how the user wants to proceed:

**[idea title]**

This note was captured without background research. Would you like me to:
1. Research the codebase now and suggest an approach
2. Just start working on it directly

Then update NOTES.md: change the status of the picked note to `✅ Done`.
