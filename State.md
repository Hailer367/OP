# State

Maintenance notes for OP. `README.md` is for the user; this file is for whoever
works on the code next. If you change behaviour, update this file in the same
commit.

## Snapshot

| | |
|---|---|
| version | `0.4.3` (`OP_VERSION` at the top of `start`) |
| repo | https://github.com/Hailer367/OP, branch `main`, public |
| size | one bash file, ~914 lines / ~40 KB, plus docs |
| runtime | bash 4.3+, `curl`, `opencode`. No daemon, no build, no package of its own |
| platform | Linux only |
| state | `~/.local/state/op` (`op.log`, `oc.pid`, `limit.flag`, `rotations`, `ask.last`, `auth.json`) |
| config | `~/.config/op/config`, patterns in `~/.config/op/patterns.conf` |

Verified end to end in a Linux sandbox with real Cloudflare WARP
(`warp-cli 2026.8.2115.0`), OpenCode `v2.4.1`, passwordless sudo, IPv6 WARP exits.

## What it does

OP runs the OpenCode TUI in a folder. A watcher tails OpenCode's log; when the
provider reports a used-up free tier, OP kills the TUI, gets a new exit IP,
verifies that the new route really works, resumes the conversation with a short
message, and reopens the TUI on the same session.

It is **supervised, not seamless**: the TUI closes and reopens. That is a design
constraint (rotation happens between sessions, not inside a live one) and is
stated in the README. Do not promise otherwise.

## Control flow

`main` → `cmd_run` (`./start`), `cmd_status`, `cmd_rotate`, `self_test`.

1. **Parse args**, then `load_config` (file), `load_patterns`, `init_state`
   (creates dirs, truncates `op.log` past `LOG_MAX_KB`).
2. **`resolve_method`** brings up egress in this order for `auto`:
   WARP (install if missing) → TOR → proxy list → otherwise plain route.
   `warp`, `tor`, `proxy`, `exec`, `none` force one method.
3. **Header**: egress, proxy URL, exit IP, log path. Warns if there is no
   terminal to draw on.
4. **Loop**, once per chat launch:
   - `token_guard` (snapshot/restore `auth.json`)
   - `export_proxy`, pick `TTY_IN` (terminal for the TUI)
   - ignore SIGINT in the supervisor, start `watch_log`, launch the TUI in the
     background with stdin bound to the terminal
   - `wait`, then stop the watcher
5. **Limit branch** (watcher wrote `limit.flag` with `rotate` or `ask`):
   - `ask` + `ASK=1` asks the user; `ask` otherwise throttled by `ASK_COOLDOWN`
   - `rot_gate` (max `ROTATE_GATE_MAX` rotations per `ROTATE_GATE_WINDOW`)
   - `hunt` (up to `ROTATE_TRIES` attempts, each verified)
   - `resume_args`: reopen the TUI with
     `--standalone --continue [--auto] --prompt <msg>`. The prompt is the resume;
     there is no separate `opencode run` blocking while the TUI is closed.
6. **Crash branch** (TUI exited non-zero, no limit): retry on the same route with
   the same prompt, three strikes in ten minutes, then stop.
7. **Exit**: `0` after a normal quit (`rc` 0/130/143), `1` on loop guard, crash
   streak or no usable route, `2` on usage errors.

## Rotation verification (`hunt`)

A candidate route must pass, in order:

1. trace answers and the exit IP **differs from the previous one**
   (full string comparison — see the open question about IPv6 `/64` below),
2. TLS handshake to `GATEWAY_URL` (`https://opencode.ai/`) succeeds,
3. unless `PROBE=0`: a throwaway session actually answers
   (`opencode run --format json --standalone --archived "reply with the word ok"`),
   deleted afterwards. This is a real model call and costs one free-tier request.

## OpenCode server

Every OpenCode process OP starts passes `--standalone`: the TUI, the probe, and
the `session list/export/delete` calls. A private server inherits the proxy
environment at launch, so it uses the route OP just verified.

The shared background service must not be used. It is started once, keeps the
environment (and therefore the old route) it was born with, and a killed session
can leave it wedged: a `run` against it then sits at `cli starting` forever. That
was the rotation that resumed into nothing.

## Egress backends

| method | how it rotates | notes |
|---|---|---|
| `warp` | disconnect → delete registration → new registration → proxy mode → connect | exit IP is IPv6 here; each rotation is a new Cloudflare registration, so it cannot be hammered |
| `tor` | restart tor (new circuit) | needs `tor` + `privoxy`; `TOR_SOCKS`, `TOR_HTTP` |
| `proxy` | next `ip:port` from `PROXY_LIST` (URL or file) | free lists are unreliable |
| `exec` | runs `EGRESS_CMD` again; it prints the proxy URL | generic escape hatch, also used by tests |
| `none` | cannot rotate | `hunt` fails, OP stops with a clear message |

WARP specifics that matter:

- Install via apt or dnf when `warp-cli` is missing (`AUTO_INSTALL=1`); needs root
  or passwordless sudo, never prompts.
- `warp-svc` is started via systemd, or directly as a daemon when systemd is not
  usable (containers).
- Both CLI syntaxes are handled: `mode proxy`/`proxy port` and the older
  `set-mode`/`set-proxy-port`; `--accept-tos` is detected, not assumed.
- A route is accepted only when the proxy answers **and** the trace says
  `warp=on|plus`. A proxy that answers is not proof that WARP carries it.

## Limits

Log lines are classified; `noop` wins over `rotate` so a context-length error
never reads as `limit reached`.

| class | default pattern | effect |
|---|---|---|
| `noop` | compaction, context length, invalid model | ignored |
| `rotate` | free-limit, usage limit, quota, credits | rotate |
| `ask` | rate limit, 429 | rotate at most once per `ASK_COOLDOWN` |

`patterns.conf` (next to the config) overrides `rotate=`, `ask=`, `noop=`; an
empty value disables that class.

## Resume

The resume is not a separate `opencode run` any more. OP reopens the TUI with
`--standalone --continue [--auto] --prompt <msg>`; the prompt is sent by the
visible chat. The old run kept the TUI closed for the whole turn and, worse, went
through the shared background server: that server still used the old route and
could be left wedged, so the run sometimes never returned and OP sat with nothing
on screen.

`RESUME_MODE=continue` sends `RESUME_MSG` (`Continue`). `last` reads the newest
session export and re-sends the last real user message; it costs two extra
`--standalone` CLI calls before the TUI reopens, and duplicates the message.
`custom` sends `RESUME_TEXT`. `AUTO_CONTINUE=1` adds `--auto` to the reopened TUI
(not just the resumed turn: the flag lasts for that session instance).

## Login

`auth.json` is copied to the state dir on every launch and restored if it
disappears. A rotation or a crash must not log the user out.

## Config

Every key can come from the environment or the config file. **The environment
wins** (`RESUME_MODE=last ./start` works even with a config file). Keys in
`RESERVED` cannot be set from the file. The config path itself is `OP_CONFIG`
(`CONFIG` is reserved to prevent recursion).

| key | default | notes |
|---|---|---|
| `EGRESS` | `auto` | `auto warp tor proxy exec none` |
| `AUTO_INSTALL` | `1` | install WARP on first run if missing |
| `FOLDER` | `$PWD` | chat folder |
| `OC` | `opencode` | binary or wrapper |
| `PROXY_LIST`, `PROXY_TRY_MAX`, `PROXY_TIMEOUT` | empty, 40, 8 | proxy backend |
| `WARP_PORT`, `WARP_WAIT` | 40000, 15 | WARP proxy port and connect wait |
| `TOR_SOCKS`, `TOR_HTTP`, `TOR_WAIT` | 9050, 8118, 45 | TOR backend |
| `EGRESS_CMD` | empty | `exec` backend; prints a proxy URL |
| `TRACE_URL`, `TRACE_CMD` | cloudflare trace, empty | `TRACE_CMD` overrides the trace (tests) |
| `GATEWAY_URL` | `https://opencode.ai/` | TLS reachability check |
| `PAT_ROTATE`, `PAT_ASK`, `PAT_NOOP` | see above | limit classes |
| `ASK_COOLDOWN` | 600 | seconds between unattended `ask` rotations |
| `ROTATE_TRIES`, `ROTATE_COOLDOWN`, `ROUTE_WAIT` | 8, 2, 15 | rotation attempts and pacing |
| `ROTATE_GATE_MAX`, `ROTATE_GATE_WINDOW` | 5, 1800 | loop guard |
| `PROBE`, `PROBE_TIMEOUT` | 1, 120 | throwaway-session check |
| `RESUME_MODE`, `RESUME_MSG`, `RESUME_TEXT` | continue, `Continue`, empty | resume |
| `AUTO_CONTINUE` | 1 | adds `--auto` to the reopened TUI |
| `ASK` | 0 | 1 = ask before rotating on a rate limit |
| `WATCH_INTERVAL` | 1 | log poll interval |
| `LOG_MAX_KB` | 1024 | activity log cap |
| `OP_STATE`, `OP_OC_LOG_DIR` | `~/.local/state/op`, `~/.local/share/opencode/log` | namespaced path overrides |

## Tests

There are four layers. All are cheap; run all of them when touching the loop.

**1. Built-in self-test** — `./start test` (offline, ~40s). It swaps only the
outside world: a fake `opencode` (writes a real limit line, answers `run`), a fake
trace that changes IP per rotation, `EGRESS=exec`. The loop itself is the real
one. Pass criteria: 3 rotations, 2 resumes, exit 1 from the loop guard.

Note: `test` runs *before* `load_config`, so it is hermetic — the config file
does not affect it.

**2. Behaviour checks** — rebuild the throwaway harness in `/tmp` with a fake
`opencode`, `EGRESS=exec`, `TRACE_CMD`, `ROTATE_GATE_MAX=1`. It is the only way
to check the pieces the self-test does not cover: noop/ask/rotate classification,
`ASK_COOLDOWN`, token guard, `RESUME_MODE=last`, log cap.

**3. TUI check in a pty** — the only way to catch terminal bugs, because a fake
TUI never reads stdin:

```bash
mkdir -p /tmp/rt && : >/tmp/rt/log/opencode.log
cat >/tmp/rt/wrap <<'E'
#!/usr/bin/env bash
t0=no; [ -t 0 ] && t0=yes
t1=no; [ -t 1 ] && t1=yes
echo "launch tty0=$t0 tty1=$t1" >>/tmp/rt/launches.txt
exec /usr/local/bin/opencode "$@"
E
chmod +x /tmp/rt/wrap
( sleep 6; printf 'abc'; sleep 10 ) | TERM=xterm-256color timeout 70 \
  script -qec "env OC=/tmp/rt/wrap OC_LOG_DIR=/tmp/rt/log FOLDER=/tmp/rt \
  STATE=/tmp/rt/state EGRESS=none ./start" /dev/null >/tmp/rt/screen.txt 2>&1
grep -c abc /tmp/rt/screen.txt   # typed text must be rendered
```

Expect `tty0=yes tty1=yes` and the typed text in the captured screen.

**4. Reopen after a rotation** — the bug this check exists for: a fake TUI that
appends one limit line to the watched log, `EGRESS=exec`, a fake trace, under
`script` (pty). After the rotation, the launches must read:

```
launch tty0=y args=--standalone
launch tty0=y args=run --format json --standalone --archived reply with the word ok
launch tty0=y args=--standalone --continue --auto --prompt Continue
```

If the third line is missing, or `--prompt` is missing, OP is again waiting on a
process instead of putting the chat back on screen.

**Live check of the real route:** `./start status`, then `./start rotate` and
confirm a genuinely different exit IP.

## Do not regress these

| bug | cause | fix |
|---|---|---|
| typing dead, UI drawn over the screen | TUI launched as a background job in a non-interactive shell → POSIX gives it `stdin=/dev/null` | bind stdin to the terminal (`TTY_IN`) |
| every launch failed in detached mode | `/dev/tty` exists without a controlling terminal; the failed redirect aborted the launch | probe by opening, fall back to `/dev/null` + one warning |
| Ctrl+C took OP (and the watcher) down | same process group, no trap | ignore SIGINT in the supervisor, start the watcher after the trap |
| Ctrl+C dead inside the chat | the chat inherited the *ignored* SIGINT | `env --default-signal=INT` for the chat |
| `Connected` matched `Disconnected` | `grep -qi connected` | word-boundary match |
| rotation loop on compaction | `limit reached` also matches context errors | `noop` class checked first |
| `Terminated` printed over the screen | foreground job killed by the watcher | run the chat in the background |
| config file silently beat env overrides | `load_config` always assigned | skip keys present in the environment |
| empty `-c ""` resume | an edit deleted `token_guard`/`resume_message`; `bash -n` and the self-test still passed | isolated behaviour checks; helpers are now covered |
| a fix killed the tool's own shell | `pkill -f <pattern>` matches the command line running it | bracket trick (`pgrep -f 'prox[y].js'`) |
| rotation ended with no TUI, `Continue` apparently sent | resume used `opencode run` against the shared background server; it kept the old route and could be wedged, so OP blocked with the TUI closed | resume is the reopened TUI's `--prompt`; all launches use `--standalone` |

## Known limitations

- Rotation is not seamless: the TUI closes for the rotation itself (3-4s on WARP
  here), then reopens with the resume prompt. The resumed turn runs in the
  visible chat, so it no longer adds to the closed window.
- `AUTO_CONTINUE=1` (default) adds `--auto` to the reopened TUI, so permissions in
  that session instance are auto-approved, not just the resumed turn.
- `RESUME_MODE=last` adds a full session export inside that window, and re-sends
  your message (duplicate turn, duplicate tokens). Default stays `continue`.
- Each WARP rotation is a new registration; Cloudflare rate-limits that. The gate
  (5 per 30 min) is the guard, not a suggestion.
- `PROBE=1` spends one real free-tier request per rotation. `PROBE=0` skips it at
  the cost of accepting a route that has not spoken to the provider yet.
- Same-IP guard compares the **full address**, not the IPv6 `/64`. Observed on
  WARP: `2a09:bac5:7490:2d2::48:fd` → `2a09:bac5:7490:2d2::48:62` is accepted as
  a rotation although the `/64` is identical. Open question: whether the free
  tier keys on the address or the prefix. Decide with real limit data before
  changing it — requiring a new `/64` would burn registrations that may be usable.
- The watcher follows the newest `*.log` by mtime and reads bytes written after
  it started. If OpenCode rotates or truncates its log mid-session, detection can
  miss lines.
- One supervised session per folder. OP does not notice another `opencode`
  running outside it, and two TUIs on the same session will fight.
- Requires root or passwordless sudo to install WARP and start `warp-svc`;
  without it the run continues on the next egress and says so once.
- Desktop-app mode, themes, menus and language files from the original project
  are deliberately not ported.

## Backlog (ranked earlier, still open)

1. `./start doctor` — one command that reports WARP installed/connected/`warp=on`,
   gateway TLS, auth present, log writable, state permissions, rotation count.
2. Geo targeting — `TARGET_COUNTRY`/`TARGET_PREFIX`, re-register until the exit
   matches, with a time budget.
3. Burn accounting — stop after N IPs the probe rejects (`IP_FIND_MAX`).
4. Other-window detection — close or warn about an OpenCode started outside OP.
5. TOR auto-install — symmetry with the WARP install.
6. Warm proxy pool — prefetch tested proxies so rotation is instant (proxy method
   only; WARP cannot be pre-registered).
7. `./start watch` — rotate for any `opencode` process, not just one OP started.
   Riskiest: it touches logs and processes it did not start.

## Working on it

- Keep it one file with no new dependencies. `bash -n` is a syntax check, not a
  test: run `./start test` and the behaviour checks as well.
- Adding a config key: default line at the top of `start`, `config.example`,
  `README.md`, the table above, and `./start --help` if it is user-facing.
- Anything that reads a log or a pid must be safe to run twice.
- Never write test limit lines into the real OpenCode log dir while a real
  session may be running; point `OC_LOG_DIR` at a temp dir instead.
- Log decisions to `op.log` (`log "..."`), keep user output on stdout short.
- Ship: commit to `main` on `Hailer367/OP` and push. Do not commit credentials;
  the repo is public and the push token is session-scoped.
