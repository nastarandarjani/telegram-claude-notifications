# Telegram notifications + auto-continue in tmux

Two things, wired together:
1. Sends a Telegram message whenever a Claude Code CLI turn ends, so you get
   pinged on your phone for long-running work — but stays silent while
   you're actively watching (attached to the `claude` tmux session, or using
   the VSCode extension).
2. When you close VSCode (specifically, when the Claude Code extension's own
   session ends), automatically spins up a detached tmux session named
   `claude` in the same directory and resumes with `claude continue` — so
   work keeps going unattended, and you get pinged on Telegram once it's
   done, per (1). When you open VSCode again, that tmux session gets killed
   unconditionally — see (4) for why.

## Components

### 1. Stop hook — `~/.claude/settings.json`

Fires on every Claude Code `Stop` event (i.e. whenever a response finishes).

```jsonc
"hooks": {
  "Stop": [{
    "hooks": [{
      "type": "command",
      "command": "vscode_active() { pgrep -u \"$USER\" -f 'vscode-server/extensions/anthropic\\.claude-code-.*resources/native-binary/claude' >/dev/null 2>&1; }; if [ ! -f ~/.claude_watching ] && ! vscode_active; then msg=$(jq -r '.last_assistant_message // \"Claude finished responding\"'); if [ ${#msg} -gt 3500 ]; then msg=\"${msg:0:3500}...\"; fi; curl -s -X POST 'https://api.telegram.org/bot<TELEGRAM_BOT_TOKEN>/sendMessage' -d 'chat_id=<TELEGRAM_CHAT_ID>' --data-urlencode \"text=$msg\" >/dev/null 2>&1; fi",
      "async": true,
      "timeout": 10
    }]
  }]
}
```

Skips sending when **either**:
- `~/.claude_watching` exists (see tmux hooks below), or
- the Claude Code VSCode extension process is running for this user (checked
  via `pgrep` for `vscode-server/extensions/anthropic.claude-code-*`).

Otherwise it POSTs the last assistant message (truncated to 3500 chars) to
your Telegram chat via the bot token embedded in that same command. Both the
token and chat ID are redacted in this repo's copy of `settings.json` —
fill in your own values locally.

### 2. Watching-marker tmux hooks — `~/.tmux.conf`

```tmux
set-hook -g client-attached 'run-shell "[ \"#{session_name}\" = claude ] && touch ~/.claude_watching || true"'
set-hook -g client-detached 'run-shell "[ \"#{session_name}\" = claude ] && { tmux list-clients -t claude 2>/dev/null | grep -q . || rm -f ~/.claude_watching; } || true"'
set-hook -g session-created 'run-shell "[ \"#{hook_session_name}\" = claude ] && [ \"#{session_attached}\" != 0 ] && touch ~/.claude_watching || true"'
```

- `client-attached` — attaching a client to an *existing* session named
  `claude` touches the marker.
- `client-detached` — detaching from `claude` removes the marker, but only
  if no clients remain attached (`tmux list-clients -t claude` comes back
  empty).
- `session-created` — covers `tmux new -A -s claude ...` when `claude`
  doesn't exist yet: `client-attached` does *not* fire in that one-step
  create-and-attach case, so this hook fills the gap. Note it uses
  `#{hook_session_name}`, not `#{session_name}` — that's the variable that
  actually resolves in this hook's context (confirmed via a throwaway test
  session; the two hooks above use `#{session_name}` instead). The
  `#{session_attached}` check distinguishes that interactive case (nonzero)
  from the SessionEnd hook's *detached* `tmux new-session -d` (reads `0` at
  creation time, confirmed via a throwaway test) — without it, every
  unattended auto-spawned session would immediately mark itself as "being
  watched," permanently suppressing the Telegram notification it exists to
  eventually trigger.

`$HOME` is shared storage across this cluster's login nodes, so the marker
file is visible regardless of which login node the Stop hook happens to run
on — a plain `who`/`tmux`/SSH-env check would be node-local and unreliable
since the cluster load-balances SSH connections across nodes.

### 3. Auto-continue hook — `~/.claude/settings.json`

Fires on `SessionEnd` (session terminates — e.g. the VSCode extension panel
closes — as opposed to `Stop`, which fires after every single turn).

```jsonc
"SessionEnd": [{
  "hooks": [{
    "type": "command",
    "command": "input=$(cat); reason=$(jq -r '.reason // \"other\"' <<<\"$input\"); cwd=$(jq -r '.cwd // empty' <<<\"$input\"); vscode_active() { pgrep -u \"$USER\" -f 'vscode-server/extensions/anthropic\\.claude-code-.*resources/native-binary/claude' >/dev/null 2>&1; }; in_claude_tmux() { [ -n \"$TMUX\" ] && [ \"$(tmux display-message -p '#S' 2>/dev/null)\" = \"claude\" ]; }; if [ -n \"$cwd\" ] && [ \"$reason\" != \"clear\" ] && [ \"$reason\" != \"resume\" ] && ! vscode_active && ! in_claude_tmux && ! tmux has-session -t claude 2>/dev/null; then tmux new-session -d -s claude -c \"$cwd\" \"claude continue\"; fi",
    "async": true,
    "timeout": 15
  }]
}]
```

Reads `cwd` (the ending session's working directory) and `reason` from the
hook's JSON stdin, then spawns the tmux session there only if **all** of
these hold:
- `reason` isn't `clear` or `resume` (those mean the session is continuing
  in some form, not actually closing — `logout`/`prompt_input_exit`/`other`
  are treated as a real close).
- The VSCode extension process isn't running anymore (`vscode_active`,
  same check as the Stop hook).
- This SessionEnd isn't itself firing *from inside* the `claude` tmux
  session (`in_claude_tmux`) — otherwise ending that session would spawn
  another one, forever.
- No session named `claude` already exists (`tmux has-session`) — avoids
  duplicate spawns if SessionEnd fires more than once in quick succession.

**On persistence across the hook's own process teardown**: Claude Code's
own hook docs warn that backgrounded processes (`nohup`, `disown`, etc.) do
not survive the triggering session's teardown, since the whole process
group gets torn down with it. Tested directly for this specific case
anyway (not just taken on faith): a `tmux new-session -d` spawned from
inside a process group, then that *entire group* sent `SIGKILL`, still left
the tmux session running and reachable afterward — every process in the
group confirmed gone via `ps`, the tmux session confirmed still listed.
tmux's server double-forks/detaches into its own session on creation,
which is specifically what makes this survive where a plain backgrounded
command would not.

### 4. Kill-on-open hook — `~/.claude/settings.json`

Fires on `SessionStart`. Solves a real staleness problem: if you close
VSCode (spawning the tmux auto-continue), then open VSCode again *while
that background session is still running*, and chat there too, the tmux
session has no way to know about those new messages — it's a separate
process with its own conversation state loaded at the moment it started.
Left alone, you'd end up with two silently-diverging conversations.

```jsonc
"SessionStart": [{
  "hooks": [{
    "type": "command",
    "command": "vscode_active() { pgrep -u \"$USER\" -f 'vscode-server/extensions/anthropic\\.claude-code-.*resources/native-binary/claude' >/dev/null 2>&1; }; in_claude_tmux() { [ -n \"$TMUX\" ] && [ \"$(tmux display-message -p '#S' 2>/dev/null)\" = \"claude\" ]; }; if vscode_active && ! in_claude_tmux; then tmux kill-session -t claude 2>/dev/null; fi",
    "async": true,
    "timeout": 10
  }]
}]
```

Unconditionally kills the `claude` tmux session (if any) whenever a VSCode
session starts — interrupting whatever it was doing, no exceptions. This
is a deliberate tradeoff: it guarantees that the *next* time you close
VSCode, the auto-continue hook (3) starts completely fresh and picks up
the latest conversation state, at the cost of losing any in-progress
background work if you happen to reopen VSCode while it's still running.

`in_claude_tmux` guards against a real self-destruction loop: spawning
`claude continue` in tmux is itself a new session starting, which would
otherwise immediately trigger this same hook and kill the session it just
created. Tested directly (not just reasoned about): the guard correctly
lets an external SessionStart (e.g. VSCode opening) kill an existing
`claude` session, while a SessionStart firing *from inside* that same
session leaves it alone.

## Net behavior

| State | Notified? |
|---|---|
| Attached to tmux session `claude` | No |
| Detached from `claude` (or never attached) | Yes |
| Running inside the VSCode Claude Code extension | No, regardless of tmux |
| Neither VSCode nor attached to `claude` | Yes |

| Event | Auto-continue in tmux? |
|---|---|
| VSCode extension session ends (VSCode/tab closed) | Yes, if `claude` session doesn't already exist |
| `/clear` or `/resume` | No — not a real close |
| Ending the `claude` tmux session itself (e.g. `/exit` inside it) | No — guarded against re-spawning itself |
| VSCode opens (any session start) while `claude` tmux session exists | Session gets killed unconditionally, freeing it up for a fresh start next close |

## Debugging checklist

- Check the marker: `ls -la ~/.claude_watching` (exists = suppressed).
- Check tmux hooks are still registered: `tmux show-hooks -g | grep -E 'client-attached|client-detached|session-created'`.
- Check VSCode detection: `pgrep -u "$USER" -f 'vscode-server/extensions/anthropic\.claude-code-.*resources/native-binary/claude'`.
- Check the auto-continue session: `tmux list-sessions | grep claude`, `tmux attach -t claude` to look inside without disturbing anything already running (detach with `Ctrl-b d` when done, not `exit`).
- If a `claude` session keeps vanishing right after you expect it to start, check whether `SessionStart`'s kill-on-open hook is the cause (it's unconditional whenever `vscode_active` is true).
- Bot token / chat ID live only in `~/.claude/settings.json` — rotate there if the bot token ever leaks.
