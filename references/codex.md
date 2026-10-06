# Codex Chats

How reopening a Codex CLI chat differs from reopening a Claude Code one. The matching rules, the Launch Clean command, and the Report in `SKILL.md` apply unchanged.

## Find The Folder

- Transcripts live at `~/.codex/sessions/<year>/<month>/<day>/rollout-<timestamp>-<session-id>.jsonl`. A chat can have more than one file, each with its session ID in the name.
- The working directory is the `cwd` inside `payload` on the `session_meta` line that opens each file and on later `turn_context` lines. Take the last one in the chat's newest file. If that folder no longer exists, ask before launching anywhere else.

## Skip A Chat That Is Already Open

A Codex chat is open when `~/.codex/thread-writer-locks/<session-id>.lock` exists. Go by the lock file, since a window that started the chat fresh has no session ID on its command line. A running `codex` process with `resume <session-id>` on its command line also means open.

Do not resume an open chat. Report it as already open. The lock stays for a short while after its window closes, so when the user says the chat is closed, wait and look again before reporting.

## Launch

Resolve the executable with `(Get-Command codex -CommandType Application | Select-Object -First 1).Source`, with the same reasons to stop as for `claude`.

Use the Launch Clean command with `$codex` in place of `$claude` and `'resume', $id` in place of `'--resume', $id`. Launching from the chat's own folder matters: started anywhere else, Codex stops to ask which directory to use.

Add no approval or sandbox flags unless the user asks for one.

## Verify

A `codex` process with `resume <session-id>` on its command line is running, and the chat's lock file exists. The process shows within a few seconds. The lock can take several times longer, so keep looking before deciding.

- With the process and no lock, report the chat as launched but not verified, with what was observed. Ask the user before stopping that process.
- With neither, tell the user the chat did not open.
