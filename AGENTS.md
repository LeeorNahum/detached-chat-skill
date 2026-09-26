# Maintaining Reopen Chat

- `SKILL.md` is the only file an agent reads at use time. Keep it a single procedure: find, skip open chats, launch clean, verify, report.
- The transcript layout, title line types, the `~/.claude/sessions` registry, and the fact that a helper session never registers there all come from observed Claude Code behavior, not documentation. Recheck them against a real transcript and a real launch when Claude Code changes how it stores or tracks sessions.
- The Windows Terminal flags follow Microsoft's Windows Terminal command-line reference. `--reloadEnvironment` exists because a tab given a command line inherits the terminal process's environment by default, and a terminal the agent starts gets the agent's.
- Test any change to the launch by reopening a real chat and confirming three things: the chat registers in `~/.claude/sessions`, the window shows color, and no transcript-saving warning appears. The agent's shell sets a no-color variable and Git variables that skip editors and prompts, so a tab that dumps its environment to a file is a quick check that none of them got through.
- Keep machine paths, session IDs, and chat titles out of the skill.
- Bump `metadata.version` on every change.
