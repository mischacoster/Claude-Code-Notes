# Reddit Post Draft

**Subreddit:** r/ClaudeAI or r/claudecode

---

**Title:** I built a /notes skill for Claude Code — capture ideas mid-session without losing your flow

**Body:**

Ever been deep in a debugging session when a completely unrelated idea hits you? "The onboarding flow needs a progress bar." Great thought. Wrong moment.

Claude Code has memory for preferences and todos for tracking your current task. But there's nothing built in for ideas that belong to *later*. You could mention them in chat, but they dissolve into context history. You could try to remember them yourself — but isn't that exactly what an AI assistant should handle?

So I built `/notes`. It works like this:

- **`/notes [idea]`** saves your thought to `NOTES.md`. One confirmation prompt the first time per session, then zero friction. Works with any permission mode.
- **`/notes -r [idea]`** does the same, but also dispatches a background Haiku agent that scans your codebase and the web. It writes a research brief with relevant files, architecture context, and a suggested approach. All async — you keep working. (Requires bypass permissions.)
- **`/notes`** (no arguments) shows your open notes. Pick one up when you're ready, and you get a structured starting point instead of a vague memory.

The interesting technical bit: background agents in Claude Code can't use the Write tool due to permission inheritance. The workaround is Bash heredocs. Also discovered that `mode: "bypassPermissions"` on the Agent tool call doesn't actually propagate to the agent's internal tool permissions — only a session-wide bypass works. The full development journey is in `DEVELOPMENT.md` for anyone building their own skills or agents.

Two files to install, one restart, and you're set.

**Repo:** github.com/mischacoster/Claude-Code-Notes

Happy to hear feedback or ideas for improvement.
