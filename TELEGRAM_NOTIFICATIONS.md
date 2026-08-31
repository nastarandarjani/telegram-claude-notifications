# Telegram "Claude finished responding" notifications

Sends a Telegram message whenever a Claude Code CLI turn ends, so you get
pinged on your phone for long-running work — but stays silent while you're
actively watching (attached to the `claude` tmux session, or using the
VSCode extension).

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
set-hook -g session-created 'run-shell "[ \"#{hook_session_name}\" = claude ] && touch ~/.claude_watching || true"'
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
  session; the two hooks above use `#{session_name}` instead).

`$HOME` is shared storage across this cluster's login nodes, so the marker
file is visible regardless of which login node the Stop hook happens to run
on — a plain `who`/`tmux`/SSH-env check would be node-local and unreliable
since the cluster load-balances SSH connections across nodes.

## Net behavior

| State | Notified? |
|---|---|
| Attached to tmux session `claude` | No |
| Detached from `claude` (or never attached) | Yes |
| Running inside the VSCode Claude Code extension | No, regardless of tmux |
| Neither VSCode nor attached to `claude` | Yes |

## Debugging checklist

- Check the marker: `ls -la ~/.claude_watching` (exists = suppressed).
- Check tmux hooks are still registered: `tmux show-hooks -g | grep -E 'client-attached|client-detached|session-created'`.
- Check VSCode detection: `pgrep -u "$USER" -f 'vscode-server/extensions/anthropic\.claude-code-.*resources/native-binary/claude'`.
- Bot token / chat ID live only in `~/.claude/settings.json` — rotate there if the bot token ever leaks.
