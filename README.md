# Telegram notifications + auto-continue in tmux

**Note**: during development, `SessionEnd` was repeatedly observed firing
for sessions other than the one actually closing, including at least
three times landing in the same second as a real close/reopen event —
the cause is still not root-caused (see "Known issue" near the end of
this file), but it's a real, recurring behavior, not a one-off. The auto-
continue spawn is built to be correct and safe regardless: a liveness
check (skips resuming a session that's still actually alive) and a
`flock`-guarded critical section around the existing-check-and-spawn
step (so two concurrent fires can't race for the same tmux session name
and silently pick the wrong winner) — both confirmed via targeted tests
and a full live close/reopen cycle that hit this exact race in the wild
and handled it correctly.

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
    "command": "input=$(cat); reason=$(jq -r '.reason // \"other\"' <<<\"$input\"); cwd=$(jq -r '.cwd // empty' <<<\"$input\"); sid=$(jq -r '.session_id // empty' <<<\"$input\"); echo \"$input\" >> ~/.claude_hook_raw.jsonl; vscode_active() { pgrep -u \"$USER\" -f 'vscode-server/extensions/anthropic\\.claude-code-.*resources/native-binary/claude' >/dev/null 2>&1; }; in_claude_tmux() { [ -n \"$CLAUDE_AUTOCONTINUE\" ]; }; va=no; vscode_active && va=yes; ict=no; in_claude_tmux && ict=yes; has_transcript=no; if [ -n \"$sid\" ]; then find ~/.claude/projects -maxdepth 2 -name \"$sid.jsonl\" 2>/dev/null | grep -q . && has_transcript=yes; fi; wants=no; [ -n \"$cwd\" ] && [ \"$reason\" != \"clear\" ] && [ \"$reason\" != \"resume\" ] && [ \"$ict\" = no ] && [ \"$has_transcript\" = yes ] && wants=yes; alive=no; if [ \"$wants\" = yes ]; then for i in 1 2 3 4 5; do sleep 2; if pgrep -u \"$USER\" -f \"resume=$sid\" >/dev/null 2>&1; then alive=yes; break; fi; done; fi; would_spawn=no; [ \"$wants\" = yes ] && [ \"$alive\" = no ] && would_spawn=yes; ex=no; spawned=no; if [ \"$would_spawn\" = yes ]; then exec 9>~/.claude_autocontinue.lock; flock -x 9; tmux has-session -t claude 2>/dev/null && ex=yes; if [ \"$ex\" = no ]; then rc=\"claude --resume=$sid\"; if tmux new-session -d -s claude -c \"$cwd\" -e CLAUDE_AUTOCONTINUE=1 \"bash -lc '$rc'\" \\; set-option -t claude remain-on-exit on; then spawned=yes; fi; fi; flock -u 9; else tmux has-session -t claude 2>/dev/null && ex=yes; fi; echo \"$(date -Is) SessionEnd fired: reason=$reason cwd=$cwd sid=$sid vscode_active=$va in_claude_tmux=$ict has_transcript=$has_transcript existing=$ex would_spawn=$would_spawn alive_after_wait=$alive spawned=$spawned\" >> ~/.claude_hook.log",
    "async": true,
    "timeout": 25
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
- `has_transcript` is `yes` — `find ~/.claude/projects -maxdepth 2 -name
  "$sid.jsonl"` finds a real, persisted transcript file for this exact
  session_id. Added after a live failure: one of the unexplained
  different-session_id `SessionEnd` fires (see "Known issue" below) won
  the spawn and the tmux pane showed `No conversation found with session
  ID: <id>` — that session_id was never a real, resumable conversation to
  begin with (confirmed: no matching `.jsonl` file exists anywhere under
  `~/.claude/projects`, only this real conversation's does). Without this
  check, one of these ephemeral ids can win the single `claude` tmux slot
  and the auto-continue fails outright for that close, with `spawned=yes`
  in the log giving no hint anything was wrong (`tmux new-session` itself
  succeeds — a dead conversation ID only breaks the `claude --resume`
  running inside it, discovered separately from `remain-on-exit` keeping
  the pane up long enough to read the error). Checked before the retry
  loop (no point waiting up to 10s for a liveness check on an id that can
  never be resumed anyway) and before acquiring the lock.
- No session named `claude` already exists (`tmux has-session`) — avoids
  duplicate spawns if SessionEnd fires more than once in quick succession.

**Liveness check, the actual mitigation for the "Known issue" below**: even
when `would_spawn` is `yes`, the hook retries `pgrep -u "$USER" -f
"resume=$sid"` up to 5 times, sleeping 2 seconds between each try (so up
to 10 seconds total, stopping early on the first hit) — VSCode's own
process always carries `--resume=<session_id>` when resuming a session,
so if a process matching that exact session becomes alive at any point
during the retry window, `alive_after_wait=yes` and the spawn is skipped,
logged, but not acted on. Only spawns when `would_spawn=yes` **and**
`alive_after_wait=no` after the full window elapses. This doesn't depend
on understanding *why* `SessionEnd` might fire on a still-live session
(see "Known issue" below) — it just refuses to act on a close claim
that's contradicted by the process table, which is the actual harm this
hook could otherwise cause. `timeout` is 25s to comfortably cover the
worst case (10s of retries plus `jq`/`pgrep` overhead).

**Why retry instead of a single check after a fixed wait**: an earlier
single-check version (wait 3s, check once) was caught missing a real
case during a live VSCode reconnect — the *new* VSCode-side process
carrying `--resume=<id>` didn't finish starting until ~4 seconds after
the triggering event, one second after the single check had already run
and concluded "not alive," so it spawned a duplicate (only cleaned up by
a lucky second `SessionStart` kill-on-open firing moments later — not
guaranteed by the design). Retested with the retry loop using a properly
isolated fake process that only appears 6 seconds in (mimicking that
delay, with margin): correctly detected as alive partway through the
retry window and skipped the spawn. A second test with a process that
never appears confirmed it still spawns normally after exhausting all 5
tries (~10s).

**Locking around the actual spawn, fixing a second, separate race**: the
liveness check above only answers "is *this* session still alive" — it
doesn't protect against two *different* `SessionEnd` fires (for two
different session_ids) racing each other for the single tmux session
name `claude`. This is a real observed case, not hypothetical: during a
live test, two `SessionEnd` fires for two completely different sessions
landed in the same second, both saw `tmux has-session -t claude` return
false (checked before either had created it), and both proceeded to spawn
— but `tmux new-session` silently fails (exit code 1, "duplicate
session") on a name that already exists, and the old code never checked
that exit code, so it logged `spawned` regardless of whether the command
actually succeeded. Evidence from that run indicated the *wrong* session
won the race and the real one silently failed to get a background copy
at all. Fixed by moving the existing-check-and-spawn into a critical
section guarded by `flock` on `~/.claude_autocontinue.lock`, and by
actually checking `tmux new-session`'s exit code (`spawned=yes` only on
real success). The liveness-check retries happen *before* acquiring the
lock (they're read-only and can take up to 10s; no reason to serialize
those), only the final check-and-create step is locked. Tested directly
by firing two `SessionEnd`s for two different fake session_ids
concurrently: exactly one now spawns (`spawned=yes`), the other correctly
detects the session already exists (`existing=yes, spawned=no`) instead
of both racing.

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
- Check `~/.claude_hook.log`: every SessionEnd fire logs one line — `SessionEnd fired: reason=... cwd=... sid=... vscode_active=... in_claude_tmux=... has_transcript=... existing=... would_spawn=... alive_after_wait=... spawned=...` — with `spawned` reflecting `tmux new-session`'s actual exit code, not just whether the hook attempted it. If there's **no `fired` line at all** for a close you expected to trigger it, the hook never ran — check whether you're looking at `tmux ls` on a different login node than the one the hook ran on (the tmux socket is node-local, unlike `~/.claude_watching` or this log file, both under `$HOME`), or whether VSCode's close path for this session didn't invoke `SessionEnd` at all. If `would_spawn=no`: check `has_transcript` first — `no` means this session_id has never been persisted as a real conversation (see "Known issue" below) and was correctly never attempted. If `would_spawn=yes` but `spawned=no`: `alive_after_wait=yes` means the liveness check correctly refused to resume a session that's still actually running; `existing=yes` with `alive_after_wait=no` means it lost the lock race to a different, concurrent `SessionEnd` fire that spawned first (also see "Known issue") — all of these are working as intended, not bugs. `~/.claude_hook_raw.jsonl` has the full unprocessed JSON payload for every fire, for whenever the extracted fields aren't enough. If a spawned tmux pane shows `No conversation found with session ID: ...`, that's the exact failure `has_transcript` now prevents — if you ever see it again, `has_transcript` is either missing from a settings.json this old, or itself has a bug.
- If a spawned session is present but shows a dead pane (`remain-on-exit` kept it around), `tmux attach -t claude` to read the error directly.
- Ruled out (tested directly, not assumed): a detached tmux session surviving after its spawning context ends does **not** require `loginctl enable-linger` on this cluster — reproduced both a raw `SIGKILL` to the whole spawning process group and a full systemd user-scope teardown (`systemd-run --user --scope`), and the tmux session survived both, consistent with `KillUserProcesses` being off (`loginctl show-user $USER --property=Linger` showing `no` didn't matter in either reproduction). If a session still vanishes with no explanation from the checks above, it's more likely the race condition described above than a linger/systemd cleanup issue.
- Bot token / chat ID live only in `~/.claude/settings.json` — rotate there if the bot token ever leaks.

## Known issue: repeated SessionEnd on a live session — cause still open, one theory retracted

While testing the fixes above live, `~/.claude_hook.log` showed the
*actual, currently open, actively-being-used* VSCode session firing
`SessionEnd` (`reason=other`) with its own real `session_id` repeatedly,
in a roughly 15-minute window (19:43-19:57 in the original investigation),
despite the conversation obviously not having ended. At the time this
looked like it might be spontaneous — a periodic background behavior
independent of anything happening in this environment — and, combined
with the (then-still-broken) `in_claude_tmux` race, it was a real risk:
each fire would have spawned a background tmux copy `--resume`-ing the
*exact same, still-open* transcript concurrently with the real session.

**A follow-up check that turned out to be invalid, corrected here rather
than left wrong**: after the burst ended (19:57:37), the following hour
showed zero further fires, which was initially presented as ruling out
"fires on every message" and "fires on a fixed interval." That check
didn't actually control for interface: partway through that hour, the
human side of this conversation switched from the VSCode extension to a
manually-created tmux session running the CLI directly — and every
message afterward, including the entire hour used for the "zero fires"
check, went through that tmux session, not the VSCode extension. So the
"zero fires" hour never actually tested whether messaging through the
VSCode extension correlates with fires; it just showed that using the
plain CLI directly doesn't produce them, which isn't the same claim and
isn't surprising either way. **The "fires after every VSCode-extension
message" hypothesis is therefore still open, not disproven.** A real test
would require reconnecting through the VSCode extension specifically and
watching `~/.claude_hook.log` while messaging that way.

**A separate, narrower observation that isn't affected by that mistake**:
the original burst's timing lines up closely with a window where
`tmux kill-session -t claude` was being run repeatedly, by hand, while
hunting down the `in_claude_tmux` race described above. Before that race
was fixed, killing the session could cause its hosted `claude` process's
own *genuine* SessionEnd to fire with the self-loop guard wrongly reading
"not in tmux" (because the session was already gone), triggering an
immediate respawn — which then got killed again, repeating the cycle. The
different session_ids seen in the burst (`3128a7a7...`, `c6a64f1b...`) are
consistent with those being the tmux-spawned copies' own session ids
(from before the `--resume=$sid` fix, when spawns used plain
`--continue`), each contributing its own SessionEnd when it was in turn
killed or replaced. This is a plausible *contributing* factor to that
specific 15-minute burst — not proven, and not a claim about what causes
`SessionEnd` to fire in general, which the VSCode-extension-message
hypothesis above is still a live candidate for.

**The liveness check stays in place regardless of which explanation (or
combination) turns out to be right** — it doesn't depend on understanding
the cause, and costs nothing when SessionEnd behaves normally.

**A properly-controlled (if small) re-test was done**: reconnected through
the VSCode extension specifically (confirmed via `TMUX` being unset and
the live process's own argv), sent a couple of real messages over a
~45-second window, and checked `~/.claude_hook.log` immediately after —
zero fires. This is real evidence against "fires on every message,"
unlike the earlier invalid hour-long check, but it's still a small sample
(one ~45s window), so "fires periodically while VSCode is active, on no
fixed short cadence" isn't ruled out by it. Considered sufficient for now
by the person doing the testing; not escalated further.

**A separate, real bug found during the reconnect part of this retest,
since fixed**: the moment VSCode reconnected, kill-on-open killed the
pre-existing tmux-hosted copy of this same session, whose own shutdown
fired a genuine SessionEnd for this session's real id. The liveness
check at the time waited 3 seconds and checked once, found no live
`--resume=<id>` process yet — because the *new* VSCode-side process
hadn't finished starting up in that window (confirmed via `ps -o
lstart`: it started ~4 seconds after the triggering event, one second
after the single check had already run) — and incorrectly spawned a
duplicate. It was cleaned up only because kill-on-open happened to fire
a second time moments later; nothing in the design guaranteed that. Fixed
by replacing the single 3-second-wait check with a retry loop (see hook
(3) above) and confirmed via isolated tests that it now catches a process
appearing partway through the window instead of missing it.

**A live close/reopen test with the retry-loop fix in place surfaced a
third, more serious real bug, since also fixed**: at the moment of
closing, *two different* SessionEnd fires landed in the same second —
one for this real conversation's session_id, and one for a completely
different, still-unexplained session_id (consistent with the open
question above: something produces SessionEnd fires for sessions other
than the one closing, and this is further evidence of it, not an
explanation). Both saw `tmux has-session -t claude` return false (checked
before either had created it) and both tried to spawn — but `tmux
new-session` on an already-existing name fails silently (confirmed
directly: exit code 1, "duplicate session"), and the code at the time
never checked that exit code, so it logged `spawned` for both regardless
of whether either actually succeeded. Evidence from the follow-up log
lines indicated the *other*, unrelated session won the race — meaning the
real conversation's background copy silently failed to exist at all for
the few minutes before VSCode was reopened, with nothing in the logs
distinguishing that from success. Fixed with a `flock`-guarded critical
section around the existing-check-and-spawn step (see hook (3) above),
and by checking `tmux new-session`'s real exit code. Tested directly by
firing two concurrent SessionEnds for two different fake session_ids:
exactly one now spawns, the other correctly backs off.

This third bug is arguably the most important one in this whole section:
it's not a safety/collision issue like the first two (nothing was
overwritten or corrupted), it's a silent **correctness** failure — the
one thing this whole hook exists to do (keep *your* conversation going in
the background) could fail with the log actively claiming it succeeded.
Combined with the still-unexplained extra session_ids, it's a reminder
that "no spawned=no in the log" was never sufficient evidence that the
feature worked, until this fix.

**A full live close/reopen test with all fixes in place (retry-loop
liveness check, lock, exit-code check) confirmed the fix directly, and
sharpened the still-open mystery**: two more concurrent
different-session_id pairs fired in the same second as real close/reopen
events (`d3338bd2...` and `8c447d50...`, alongside this real session's own
id both times) — the lock correctly let exactly one of each pair spawn
and made the other back off honestly (`existing=yes, spawned=no`), with
no silent duplication either way, including the case where the *real*
session lost the race the first time and correctly got nothing, and won
it the second time and correctly got a real background copy. Final state
after both cycles: no leftover tmux session, exactly one live process
(the real, VSCode-side one), no duplicates.

This makes **three separate occasions** (`49f6ec47...` on an earlier
reconnect, now these two) where an unexplained, different session_id
fired `SessionEnd` at almost the exact same moment as a real close/reopen
event. That's a much more specific pattern than "random every 20-70s" or
"every message sent" — both considered and moved past earlier in this
section — and points toward something in VSCode's own close/reopen
mechanics specifically (a companion process, a telemetry/diagnostic
session, some internal restart of its own) rather than a periodic
background timer or per-message behavior. Still not root-caused, and not
pursued further for now, but worth keeping in mind as the most
specific lead if this is ever revisited.

**A key fact about these mystery session_ids, learned the hard way**:
one of them (a fourth occurrence, `6241a161...`) won the spawn race on a
real close, and when the user attached to the resulting tmux pane it
showed `No conversation found with session ID: 6241a161-...` — a hard
CLI error, not a hang or a silent success. Checked directly: no `.jsonl`
transcript file exists anywhere under `~/.claude/projects` for that id,
or for any of the other mystery ids from earlier in this section
(`49f6ec47`, `d3338bd2`, `8c447d50`, `3128a7a7`, `c6a64f1b`) — only this
real conversation's own id has ever been persisted. So whatever produces
these ids, they are not real, resumable conversations and never will be;
`claude --resume=<one of them>` is guaranteed to fail. This doesn't
explain *why* they fire `SessionEnd`, but it does mean the practical
fix doesn't need to wait on that explanation: hook (3) now requires a
real transcript file to exist for a session_id before ever trying to
resume it (see `has_transcript` above), so one of these ids can no
longer silently consume the single tmux slot and break auto-continue for
the real, closing session. `remain-on-exit` (added earlier for a
different reason) is what made this error visible at all instead of the
pane just quietly vanishing — worth remembering as a generally useful
property of that setting.
