---
name: "detached-chat"
description: "Use when the user asks to reopen, resume, bring back, or open again one or more past chats, sessions, or threads, usually by name or title, even when they do not say Claude Code, Codex, or session. Also use when the user asks for work to go to a chat of its own instead of a subagent: work they say must keep running after the current session closes, work they ask to watch or steer in its own window, or a past chat they want to pick up new instructions. Only for Claude Code CLI and Codex CLI sessions, so skip it when the user names any other chat app or website. Reopens past Claude Code and Codex CLI sessions by title, each in its own new terminal window from its original folder, and hands work to a Claude Code window chat or background chat that outlives the current session, with a way to watch it and to tell how it ended."
compatibility: "Claude Code CLI 2.1.257 or later for background chats. Codex CLI 0.160 or later to reopen Codex chats. Opening a window requires Windows with Windows Terminal and PowerShell 7."
metadata:
  author: "Leeor Nahum"
  version: "2.2.0"
---

# Detached Chat

A chat here is a Claude Code CLI or Codex CLI session, which the user may also call a session or a thread, not an artifact or document with a similar name. When the user names any other chat app or website, this skill does not apply. A detached chat runs on its own, not as part of the session that started it.

| The user wants | Do |
| --- | --- |
| A past chat back in front of them | Reopen it: Find Each Session, Skip Chats That Are Already Open, Launch Clean, Verify |
| Work in a chat of its own, new or continued from a past chat | Hand Off Work |
| To watch or type into a background chat | Open A Window On A Background Chat |

Reopening covers both CLIs, and the sections below give the Claude Code specifics. As soon as Find Each Session shows that a chat is a Codex one, read [Codex chats](references/codex.md) and use its folder, open check, launch arguments, and verification in their place.

## Find Each Session

- When the user does not say which CLI the chat belongs to, search the titles of both and let the best match decide. Ask when a Claude Code chat and a Codex chat match equally well.
- Codex titles are the `thread_name` values in `~/.codex/session_index.jsonl`, one JSON line per naming, each with the chat's session ID in `id`. The last line for an `id` holds its current title.
- Claude Code transcripts live at `~/.claude/projects/<encoded-folder>/<session-id>.jsonl`. The file name is the session ID.
- Claude Code titles are JSON lines with `"type":"custom-title"` (field `customTitle`, set by the user) or `"type":"ai-title"` (field `aiTitle`, generated). A rename appends a new line. The current title is the last custom title, or the last AI title when the chat has no custom title.
- Match on meaning, not exact text. The user may reorder words, drop an emoji or a suffix, or use a shorter name than the title. Search every project folder, since the chat may have started anywhere.
- Skip the current session. Its ID is in the `CLAUDE_CODE_SESSION_ID` variable of the agent's shell, and its own title often echoes the request.
- When more than one session fits, take the strongest title match. Break a tie by what the user said about the chat, then by the most recently modified, and name the others on that chat's report line. Ask instead when nothing separates them but the modified time.
- The working directory of a Claude Code chat is the last `cwd` field in the transcript. Never decode it from the folder name, which turns every separator, space, and dot into a hyphen and cannot be reversed. If that folder no longer exists, ask before launching anywhere else.

## Skip Chats That Are Already Open

Each running Claude Code session registers `~/.claude/sessions/<pid>.json`, whose `sessionId` field names the chat it holds. A chat is open when one of those files names its session ID and the process with that `pid` is still running as `claude`. Files whose process is gone, or whose `pid` now belongs to another program, are stale and mean nothing. Never resume an open chat, since two windows on one transcript split the conversation. Report it as already open. When its file says `"kind":"bg"`, it is a background chat, and the way to put it in front of the user is Open A Window On A Background Chat.

A helper session never registers there, so also look for a running `claude` process with `--resume <session-id>` on its command line. A match with no registry file is the chat open as a helper of some agent's session, not saving its transcript. Report it as open but not saving, and ask the user before closing it and reopening it clean, since they may be typing in it.

## Launch Clean

Anything the agent starts inherits its shell's environment, which marks the process as a helper of the current session and tunes tools for an agent rather than a person. A chat started in a window that inherits it runs as a helper: it stops saving its transcript and loses color. Launch with a fresh environment built from the user's saved variables, never by deleting known names from the agent's own, which misses whatever the harness adds next.

A Windows Terminal tab given a command line inherits the terminal process's environment, which comes from whoever launched it, and `--reloadEnvironment` makes it build a fresh one instead.

Before launching, resolve the executable with `(Get-Command claude -CommandType Application | Select-Object -First 1).Source`. Stop and tell the user, instead of improvising a different launch, when it finds nothing, when it resolves to a script shim rather than an `.exe`, or when `$cwd` contains a semicolon, which Windows Terminal reads as a command separator.

Then run this through PowerShell once per chat, with `$claude`, `$cwd`, and `$id` set, because Git Bash mangles the quoting:

```powershell
Start-Process wt -ArgumentList @('-w', 'new', 'nt', '--reloadEnvironment', '-d', "`"$cwd`"", "`"$claude`"", '--resume', $id)
```

Add no permission flags unless the user asks for one. Claude Code sets the resumed chat's mode from the chat and the user's settings.

## Verify

Within a few seconds of a clean launch, a new `~/.claude/sessions/<pid>.json` names the chat's session ID, with that process running. A helper of the agent's session never registers there. When the file is missing but a `claude --resume <id>` process runs, report the chat as launched but not verified, with what was observed, and diagnose no further. Ask the user before stopping that process, since they may already be typing in it. Registration cannot show color or the absence of the transcript warning, so the report asks the user to glance at the window.

## Hand Off Work

A subagent of the current session stays the default for delegated work, however long it runs and whatever state its folder is in. Hand work to a chat of its own only when the user asks for one in this conversation: work they say must keep running after this session closes or is stopped, work they ask to watch or steer, or a past chat they want to pick up new instructions. A project instruction or a plan that calls for one, or the agent's own judgment that the work is long, in a folder with uncommitted changes, or worth watching, is a reason to ask the user first, never to launch. A window opening on the user's screen that they did not ask for is the failure this rule prevents. `claude -p` is not a chat of its own: `--bg` rejects it, and a `-p` run the agent starts belongs to the agent's shell.

Handing off work is Claude Code only. When the chat that should take the work is a Codex one, say so and offer to reopen it for the user instead.

### Write The Task File

The chat starts with nothing from this session. Write the whole task to a file, by absolute path, in a folder that outlives this session, never the session's scratch folder. It says the folder and branch the work belongs on, what the chat may commit, push, or merge, what finished means, what to do with a decision only the user can make, and the file its final report goes to.

The prompt is then one line, `Read <task file> and do what it says.` Stop and ask when the path, or a name the user chose, holds a semicolon or a double quote, which break the launch.

Use the model, effort, and name already chosen for the work. When none was chosen, leave `--model` and `--effort` off and name the chat after the task.

Add no permission flags unless the user asks for one. A chat that reaches a permission prompt its mode does not answer waits there.

### Choose Where It Runs

| | Window chat | Background chat |
| --- | --- | --- |
| Runs | In its own terminal window, on screen from the start | Under Claude Code's supervisor, with no window |
| Lasts | Until the user closes the window or the machine shuts down | Through closed terminals and sleep. A shutdown stops it, and attaching restarts it where it left off |
| Edits land | In the folder itself, on the branch checked out there | By default in a Git repository, in a worktree of its own under `.claude/worktrees/`, on its own branch |
| Use for | Work that changes the checkout in place, switches branches, merges, or releases | Long or parallel work whose result can arrive as a branch, or that edits no repository |

A background chat that enters a worktree commits there and pushes its branch when the repository has a remote, and never merges or pushes the default branch. The task file's Git instructions take precedence over that. It edits the folder itself instead when the folder is not a Git repository, is already a linked worktree, or belongs to a repository whose settings set `worktree.bgIsolation` to `"none"`. Check which applies before choosing. Never add that setting to change where a chat's edits land. Use a window chat.

### Continue A Past Chat

On either route, Find Each Session and Skip Chats That Are Already Open come before any launch. Only a chat that is not open is started.

When it is open, do not resume it. When the harness can message other sessions on the machine, send it the one-line prompt, and report it as continued only when the send says it was delivered, not held for the user's approval in that chat. Otherwise report it as already running and ask the user.

Continue a past chat in a window when work it left uncommitted sits in its folder, since a background chat may move to a worktree that does not have it.

### Start Or Continue A Window Chat

Use the Launch Clean command with these in place of `'--resume', $id`, then Verify.

- A new chat: `'--session-id', $id, '-n', "`"$name`"", "`"$prompt`""`, with `$id` a new UUID and `$cwd` the folder the work belongs in. Add `'--model', $model, '--effort', $effort` before the prompt when they were chosen.
- A past chat that is not open: `'--resume', $id, "`"$prompt`""`.

A new chat runs with `--session-id`, so Verify looks for the registry file naming `$id`, or failing that a `claude` process with `$id` on its command line. With neither, report it as failed to start.

### Start Or Continue A Background Chat

Run one of these from the folder the work belongs in, which for a past chat is its last `cwd`:

```bash
claude --bg -n "<name>" --model <model> --effort <level> "Read <task file> and do what it says."
claude --bg --resume <full session id> "Read <task file> and do what it says."
```

The command prints `backgrounded`, the chat's short ID, and its name, sometimes after a line about starting the background service. Then check what it started:

- `Workspace not trusted` means no chat started. Trusting a folder is the user's to do.
- Find the short ID in `claude agents --json --all`. A new chat is listed under its name.
- For a past chat, compare the listed `sessionId` with the one asked for. The same ID is a continuation. A different one is a copy, a `note:` line in the output says why, and the report calls it a copy. A matching name proves neither. A name or a bare `--resume` in place of the full session ID always starts a copy.

`claude attach`, `claude logs`, `claude stop`, and `claude rm` take the short ID. `claude agents` lists every chat.

### Open A Window On A Background Chat

Use the Launch Clean command with `'attach', $shortId` in place of `'--resume', $id`. The short ID is the `id` beside the chat's `sessionId` in `claude agents --json --all`, which also gives its `cwd`. Read it there, never by cutting the session ID short.

Verify does not apply to this window. The check is a running `claude attach <short id>` process. Closing the window leaves the chat running.

### Learn How It Ended

A chat that has stopped working is not always finished.

- Where the harness can message other sessions on the machine, subscribe once to the chat's idle notice. It lasts 12 hours and never arrives from a chat that refuses messages.
- When the notice arrives, read the chat's entry in `claude agents --json --all`. A background chat has `state`: `working`, `blocked`, `done`, `failed`, or `stopped`. A window chat has `status`: `busy`, `waiting`, or `idle`. `waitingFor` names what a blocked or waiting chat needs, such as a permission prompt nobody answered.
- With no way to subscribe, read that entry when the user asks, or at a pace that fits the work. Never message the chat to ask whether it is done.
- A window chat that has exited has no entry. Read the report file, and when there is none, say the chat ended with its result unverified.
- Call it finished only after reading the report file the task named. Work that outlives this session has no chat left to notify, so that file is how the user learns the result.

### Stop Or Remove A Chat

Ask before `claude stop <short id>` on a chat the user may be typing in. Stopping keeps the conversation, and `claude rm <short id>` takes a chat off the list while its transcript stays.

## Report

One line per reopened chat: its current title, which CLI it belongs to, its folder, any other sessions that also matched, and whether it was opened, was already open, was open but not saving, was launched but not verified, or was not found.

One line per handed-off chat: its name, where it runs (a window, or the background with its short ID), its folder, the task file, the report file, and whether it was started, was continued, was started as a copy, was already running with the instructions delivered or not, or failed to start and why. For a background chat, add that `claude attach <short id>` opens it. When it ends, add how it ended and where the work is.
