---
name: "reopen-chat"
description: "Use when the user asks to reopen, resume, bring back, or open again one or more past chats or sessions, usually by name or title, even when they do not say Claude Code or session. Only for Claude Code CLI sessions, so skip it when the user names any other chat app or website. Finds each past Claude Code CLI session by its title and relaunches it in its own new terminal window, from its original folder, as a normal session the user can type into."
compatibility: "Windows with Windows Terminal and PowerShell 7."
metadata:
  author: "Leeor Nahum"
  version: "1.2.0"
---

# Reopen Chat

A chat here is a past Claude Code CLI session, not an artifact or document with a similar name. When the user names any other chat app or website, this skill does not apply. Reopening one means running `claude --resume <session-id>` in a new Windows Terminal window.

## Find Each Session

- Transcripts live at `~/.claude/projects/<encoded-folder>/<session-id>.jsonl`. The file name is the session ID.
- Titles are JSON lines with `"type":"custom-title"` (field `customTitle`, set by the user) or `"type":"ai-title"` (field `aiTitle`, generated). A rename appends a new line. The current title is the last custom title, or the last AI title when the chat has no custom title.
- Match on meaning, not exact text. The user may reorder words, drop an emoji or a suffix, or use a shorter name than the title. Search every project folder, since the chat may have started anywhere.
- Skip the current session. Its ID is in the `CLAUDE_CODE_SESSION_ID` variable of the agent's shell, and its own title often echoes the request.
- When more than one session fits, take the strongest title match. Break a tie by what the user said about the chat, then by the most recently modified, and name the others on that chat's report line. Ask instead when nothing separates them but the modified time.
- The working directory is the last `cwd` field in the transcript. Never decode it from the folder name, which turns every separator, space, and dot into a hyphen and cannot be reversed. If that folder no longer exists, ask before launching anywhere else.

## Skip Chats That Are Already Open

Each running Claude Code session registers `~/.claude/sessions/<pid>.json`, whose `sessionId` field names the chat it holds. A chat is open when one of those files names its session ID and the process with that `pid` is still running as `claude`. Files whose process is gone, or whose `pid` now belongs to another program, are stale and mean nothing. Never resume an open chat, since two windows on one transcript split the conversation. Report it as already open.

A helper session never registers there, so also look for a running `claude` process with `--resume <session-id>` on its command line. A match with no registry file is the chat open as a helper of some agent's session, not saving its transcript. Report it as open but not saving, and ask the user before closing it and reopening it clean, since they may be typing in it.

## Launch Clean

Anything the agent starts inherits its shell's environment, which marks the process as a helper of the current session and tunes tools for an agent rather than a person. A resumed chat that inherits it runs as a helper: it stops saving its transcript and loses color. Launch with a fresh environment built from the user's saved variables, never by deleting known names from the agent's own, which misses whatever the harness adds next.

A Windows Terminal tab given a command line inherits the terminal process's environment, which comes from whoever launched it, and `--reloadEnvironment` makes it build a fresh one instead.

Before launching, resolve the executable with `(Get-Command claude -CommandType Application | Select-Object -First 1).Source`. Stop and tell the user, instead of improvising a different launch, when it finds nothing, when it resolves to a script shim rather than an `.exe`, or when `$cwd` contains a semicolon, which Windows Terminal reads as a command separator.

Then run this through PowerShell once per chat, with `$claude`, `$cwd`, and `$id` set, because Git Bash mangles the quoting:

```powershell
Start-Process wt -ArgumentList @('-w', 'new', 'nt', '--reloadEnvironment', '-d', "`"$cwd`"", "`"$claude`"", '--resume', $id)
```

Add no permission flags unless the user asks for one. Claude Code sets the resumed chat's mode from the chat and the user's settings.

## Verify

Within a few seconds of a clean launch, a new `~/.claude/sessions/<pid>.json` names the chat's session ID, with that process running. A helper of the agent's session never registers there. When the file is missing but a `claude --resume <id>` process runs, report the chat as launched but not verified, with what was observed, and diagnose no further. Ask the user before stopping that process, since they may already be typing in it. Registration cannot show color or the absence of the transcript warning, so the report asks the user to glance at the window.

## Report

One line per chat: its current title, its folder, any other sessions that also matched, and whether it was opened, was already open, was open but not saving, was launched but not verified, or was not found.
