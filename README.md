# Telegram notifications + auto-continue in tmux

**⚠️ Known issue, mitigated but not root-caused**: a still-actively-used
VSCode session was observed repeatedly emitting `SessionEnd` events
(`reason=other`, its own real `session_id`) roughly every 20-70 seconds
despite nothing actually ending. Left unmitigated, this would make the
auto-continue spawn below resume the *exact same, still-live* session
concurrently with the real one — two processes writing to the same
transcript at once. The spawn now includes a liveness check (see hook (3)
below) that closes this specific risk regardless of why the repeated
firing happens, but the firing itself is still not understood. See "Known
issue: repeated SessionEnd on a live session" near the end of this file.

Two things, wired together:
1. Sends a Telegram message whenever a Claude Code CLI turn ends, so you get
   pinged on your phone for long-running work — but stays silent while
   you're actively watching (attached to the `claude` tmux session, or using
   the VSCode extension).
2. When you close VSCode (specifically, when the Claude Code extension's own
   session ends), automatically spins up a detached tmux session named
   `claude` in the same directory and resumes with `claude --continue` — so
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
    "command": "input=$(cat); reason=$(jq -r '.reason // \"other\"' <<<\"$input\"); cwd=$(jq -r '.cwd // empty' <<<\"$input\"); sid=$(jq -r '.session_id // empty' <<<\"$input\"); echo \"$input\" >> ~/.claude_hook_raw.jsonl; vscode_active() { pgrep -u \"$USER\" -f 'vscode-server/extensions/anthropic\\.claude-code-.*resources/native-binary/claude' >/dev/null 2>&1; }; in_claude_tmux() { [ -n \"$CLAUDE_AUTOCONTINUE\" ]; }; va=no; vscode_active && va=yes; ict=no; in_claude_tmux && ict=yes; ex=no; tmux has-session -t claude 2>/dev/null && ex=yes; would_spawn=no; [ -n \"$cwd\" ] && [ \"$reason\" != \"clear\" ] && [ \"$reason\" != \"resume\" ] && [ \"$ict\" = no ] && [ \"$ex\" = no ] && would_spawn=yes; alive=no; if [ \"$would_spawn\" = yes ] && [ -n \"$sid\" ]; then sleep 3; pgrep -u \"$USER\" -f \"resume=$sid\" >/dev/null 2>&1 && alive=yes; fi; echo \"$(date -Is) SessionEnd fired: reason=$reason cwd=$cwd sid=$sid vscode_active=$va in_claude_tmux=$ict existing=$ex would_spawn=$would_spawn alive_after_wait=$alive\" >> ~/.claude_hook.log; if [ \"$would_spawn\" = yes ] && [ \"$alive\" = no ]; then if [ -n \"$sid\" ]; then rc=\"claude --resume=$sid\"; else rc=\"claude --continue\"; fi; tmux new-session -d -s claude -c \"$cwd\" -e CLAUDE_AUTOCONTINUE=1 \"bash -lc '$rc'\" \\; set-option -t claude remain-on-exit on; echo \"$(date -Is) SessionEnd: spawned claude tmux session (cwd=$cwd sid=$sid)\" >> ~/.claude_hook.log; fi",
    "async": true,
    "timeout": 15
  }]
}]
```

Reads `cwd`, `reason`, and `session_id` from the hook's JSON stdin, and
also appends the full raw JSON payload to `~/.claude_hook_raw.jsonl`
unconditionally (in case a future fire needs fields beyond the ones this
hook already extracts). Logs a `SessionEnd fired:` line to
`~/.claude_hook.log` recording all of these plus a `would_spawn` verdict
and an `alive_after_wait` verdict — this runs on *every* SessionEnd, not
just ones that go on to spawn, specifically so a "nothing happened and I
don't know why" report has a paper trail instead of requiring blind
re-diagnosis. `would_spawn` is computed from all of these holding:
- `reason` isn't `clear` or `resume` (those mean the session is continuing
  in some form, not actually closing — `logout`/`prompt_input_exit`/`other`
  are treated as a real close).
- This SessionEnd isn't itself firing *from inside* the `claude` tmux
  session (`in_claude_tmux`) — otherwise ending that session would spawn
  another one, forever.
- No session named `claude` already exists (`tmux has-session`) — avoids
  duplicate spawns if SessionEnd fires more than once in quick succession.

**Liveness check, the actual mitigation for the "Known issue" below**: even
when `would_spawn` is `yes`, the hook waits 3 seconds and then checks
`pgrep -u "$USER" -f "resume=$sid"` — VSCode's own process always carries
`--resume=<session_id>` when resuming a session, so if a process matching
that exact session is still alive after the wait, `alive_after_wait=yes`
and the spawn is skipped, logged, but not acted on. Only spawns when
`would_spawn=yes` **and** `alive_after_wait=no`. This doesn't depend on
understanding *why* `SessionEnd` might fire on a still-live session (see
"Known issue" below) — it just refuses to act on a close claim that's
contradicted by the process table, which is the actual harm this hook
could otherwise cause. Tested directly: a fake process holding
`--resume=<id>` in its argv correctly blocks the spawn (confirmed no tmux
session or socket is even created); with no such process, it spawns
normally.

**`in_claude_tmux` no longer queries tmux live** — it used to be
`[ -n "$TMUX" ] && [ "$(tmux display-message -p '#S')" = claude ]`, which
has a real race: if the `claude` tmux session has *just* been killed (by
this same fire, by `SessionStart`'s kill-on-open, or by hand), that query
fails because the session no longer exists, so it wrongly evaluates to
"not in tmux" — bypassing the self-loop guard and causing an immediate,
spurious respawn on every single kill. Caught directly: manually killing
the `claude` session repeatedly produced an instant respawn every time.
Fixed by setting a plain environment variable (`-e CLAUDE_AUTOCONTINUE=1`
on the `tmux new-session` call, see below) and checking that instead —
it's inherited by the whole process tree at fork time and survives the
session dying, unlike a live tmux query.

**The spawn uses `--resume=$session_id` instead of `claude --continue`**
— `session_id` comes from the same hook JSON stdin as `cwd`/`reason`.
`--continue` resumes "the most recent conversation in the current
directory," which is ambiguous with more than one session open in the
same repo; `--resume=<id>` targets the exact session that fired this
`SessionEnd`, whatever else might be running there. (Falls back to
`--continue` only if `session_id` is somehow absent from the payload.)

`vscode_active` is still computed and logged for every fire, but is
**not** a gating condition here (a change from an earlier version — see
below for why).

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

**`bash -lc 'claude --continue'` instead of a bare `claude --continue`**: found
the hard way — `claude` is a user-local install (`~/.local/bin/claude`),
which is only on `PATH` once `~/.bashrc` has been sourced. tmux runs a
`new-session` command via the default shell *non-interactively and
non-login*, so it does **not** source `~/.bashrc`, so `claude` was not
found. That failure was silent and fast: the pane's only command exited
immediately with "command not found," and since it was the session's only
pane, the whole tmux session died within a second or two — before
`tmux ls` from anywhere would ever show it, and before it could reach the
point of resuming (so no Telegram notification either). Confirmed directly
by reproducing it: spawning the old command under a `PATH` stripped down
to just `/usr/bin`, the session vanished near-instantly; the same
reproduction with `bash -lc '...'` left a real, running `claude` process in
the pane. `bash -lc` forces a login shell, which sources the rc files and
fixes `PATH` regardless of what environment the hook itself inherited.
`set-option -t claude remain-on-exit on` is a second line of defense: if
`claude --continue` ever exits for some *other* reason (crash, bad
resume state, etc.), the pane stays around showing the error instead
of the whole session disappearing with no trace. The trailing `echo ... >>
~/.claude_hook.log` records every spawn attempt (timestamp, cwd, reason)
so a future failure has a paper trail instead of requiring a fresh
from-scratch investigation.

**Why `vscode_active` is logged but not gated on here (a change from an
earlier version)**: it used to also require `vscode_active` to be false
before spawning, on the theory that the VSCode extension process being
gone was a good proxy for "the user is really done." In practice this
blocked real closes: VSCode's remote-SSH server-side process can keep
running for a while after a UI disconnect (it supports reconnecting), so
`vscode_active` stayed `yes` well past the point where `SessionEnd` had
already correctly fired with a real close reason. Caught directly via the
log this hook now writes: a real disconnect logged a `SessionEnd fired`
line with `vscode_active=yes`, ended up on the `if` condition's wrong side
with the old code, and never spawned — with no retry, since `SessionEnd`
is a one-shot event. Dropping `vscode_active` from the condition (while
still logging it, since it's useful context) fixes this; it doesn't
reopen the "spawn while the user is still actively working" problem the
check was meant to prevent, because hook (4) below already kills any
spawned session the instant a real new session actually starts.

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
    "command": "vscode_active() { pgrep -u \"$USER\" -f 'vscode-server/extensions/anthropic\\.claude-code-.*resources/native-binary/claude' >/dev/null 2>&1; }; in_claude_tmux() { [ -n \"$CLAUDE_AUTOCONTINUE\" ]; }; va=no; vscode_active && va=yes; ict=no; in_claude_tmux && ict=yes; killed=no; if [ \"$va\" = yes ] && [ \"$ict\" = no ]; then tmux has-session -t claude 2>/dev/null && killed=yes; tmux kill-session -t claude 2>/dev/null; fi; echo \"$(date -Is) SessionStart fired: vscode_active=$va in_claude_tmux=$ict killed=$killed\" >> ~/.claude_hook.log",
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
`claude --continue` in tmux is itself a new session starting, which would
otherwise immediately trigger this same hook and kill the session it just
created. Tested directly (not just reasoned about): the guard correctly
lets an external SessionStart (e.g. VSCode opening) kill an existing
`claude` session, while a SessionStart firing *from inside* that same
session leaves it alone.

Every fire is logged unconditionally (`SessionStart fired:
vscode_active=... in_claude_tmux=... killed=...`), for the same reason as
hook (3)'s logging: VSCode's remote-SSH disconnect sequence can be noisy
(several `SessionStart`/`SessionEnd` cycles in quick succession, likely
reconnect attempts), and this is what makes it possible to see, after the
fact, whether this hook is what killed a session you expected to survive.

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
- If a `claude` session keeps vanishing right after you expect it to start, check `~/.claude_hook.log` for a `SessionStart fired: ... killed=yes` line around that time — that's this cluster's actual observed failure mode: VSCode's disconnect can trigger a noisy flurry of `SessionStart`/`SessionEnd` cycles (likely reconnect attempts), and any `SessionStart` with `vscode_active=yes` kills the `claude` session unconditionally, including one that was just spawned moments earlier.
- Check `~/.claude_hook.log`: every SessionEnd fire logs a `SessionEnd fired: reason=... cwd=... sid=... vscode_active=... in_claude_tmux=... existing=... would_spawn=... alive_after_wait=...` line unconditionally, plus a second `spawned` line if it actually spawned. If there's **no `fired` line at all** for a close you expected to trigger it, the hook never ran — check whether you're looking at `tmux ls` on a different login node than the one the hook ran on (the tmux socket is node-local, unlike `~/.claude_watching` or this log file, both under `$HOME`), or whether VSCode's close path for this session didn't invoke `SessionEnd` at all. If there's a `fired` line with `would_spawn=yes` but no `spawned` line, check `alive_after_wait`: `yes` means the liveness check correctly refused to resume a session that's still actually running (see "Known issue" below); if `alive_after_wait=no` and it still didn't spawn, something else is wrong and worth a fresh look. `~/.claude_hook_raw.jsonl` has the full unprocessed JSON payload for every fire, for whenever the extracted fields aren't enough.
- If a spawned session is present but shows a dead pane (`remain-on-exit` kept it around), `tmux attach -t claude` to read the error directly.
- Ruled out (tested directly, not assumed): a detached tmux session surviving after its spawning context ends does **not** require `loginctl enable-linger` on this cluster — reproduced both a raw `SIGKILL` to the whole spawning process group and a full systemd user-scope teardown (`systemd-run --user --scope`), and the tmux session survived both, consistent with `KillUserProcesses` being off (`loginctl show-user $USER --property=Linger` showing `no` didn't matter in either reproduction). If a session still vanishes with no explanation from the checks above, it's more likely the race condition described above than a linger/systemd cleanup issue.
- Bot token / chat ID live only in `~/.claude/settings.json` — rotate there if the bot token ever leaks.

## Known issue: repeated SessionEnd on a still-live session (mitigated, not root-caused)

While testing the fixes above live, `~/.claude_hook.log` showed the
*actual, currently open, actively-being-used* VSCode session firing
`SessionEnd` (`reason=other`) with its own real `session_id` repeatedly —
roughly every 20-70 seconds — despite the conversation obviously not
having ended (it kept responding immediately after each fire). This is
not the VSCode-reconnect-flapping pattern described elsewhere in this
file (that involves distinguishable `SessionStart`/`SessionEnd` pairs from
a session actually restarting); this was the *same* `session_id` firing
`SessionEnd` over and over while clearly still alive.

Before the liveness check existed, each of these fires would have spawned
a background tmux copy that ran `--resume=<that same session_id>` — i.e.
a second live process resuming the *exact* transcript the real, still-open
session was using, concurrently. That's a real risk of two processes
writing to the same conversation state at once, not just a wasted spawn.
Caught directly by inspecting the process tree (`pstree`/`pgrep`) and
finding two `claude` processes both tied to the same session id at the
same time, followed by confirming via the log that the repeated fires
shared that id.

The cause of the repeated `SessionEnd` firing itself is **still not
understood**. It doesn't correlate with any tmux activity in this
environment (killing/spawning tmux sessions was ruled out as the trigger:
the fires continued at their own cadence independent of that), and other,
*different* session_ids were also observed firing SessionEnd around the
same times as the main session's own id — so this isn't specific to one
misbehaving tab. Asked Claude Code's own documentation (via a
`claude-code-guide` research pass) whether background prefetching, or any
other internal mechanism, spawns auxiliary sessions with their own
session_ids that would trigger these hooks, and whether the hook payload
has any field to distinguish a main interactive session from an internal
one: **the docs don't say**. Confirmed fields are `session_id`, `prompt_id`,
`transcript_path`, `cwd`, `scratchpad_dir`, `permission_mode`,
`hook_event_name`, plus `agent_id`/`agent_type` for subagents; the
`reason` field's possible values and meanings (including what `other`
covers) aren't documented at all, and there's no documented way to filter
hooks to "real" top-level sessions only. This is a genuine documentation
gap, not something resolvable from this side alone.

**Given the above, the mitigation in place is the actual fix, not a
workaround pending a real one**: the liveness check (see hook (3) above)
means the spawn's correctness no longer depends on understanding why
`SessionEnd` fires — it only acts when the process table confirms the
session is actually gone. The spawn is therefore re-enabled. If the
repeated-firing cause is ever found (e.g. filing a `/feedback` request
for the missing docs, or checking Claude Code's `--debug --debug-to-stderr`
output), it would mostly matter for reducing log noise (`~/.claude_hook.log`
and `~/.claude_hook_raw.jsonl` will keep accumulating `would_spawn=yes
alive_after_wait=yes` entries for every spurious fire), not for closing a
remaining safety gap.
