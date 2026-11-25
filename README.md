# OP

OpenCode that does not stop when the free tier does.

OP runs the OpenCode TUI in a folder. When the provider answers with a limit, it
swaps the exit IP, checks the new route can actually reach the gateway, proves it
with a throwaway session, and resumes the same chat.

One file, `./start`. bash + curl + opencode. Linux only, no build step, no daemon,
no dependencies to install.

## Use

```bash
./start                 # run the chat in the current folder
./start status          # route, exit IP, paths
./start rotate          # new exit IP, verified
./start test            # offline self-test of the whole loop
./start --help
```

Flags: `--folder DIR`, `--egress auto|warp|tor|proxy|exec|none`, `--no-probe`,
`--dry-run`.

## Egress

`auto` picks the first method that comes up: WARP, then TOR, then a proxy list.

| method | needs | rotation |
|---|---|---|
| `warp` | `warp-cli` + `warp-svc`, root or passwordless sudo | re-register, reconnect |
| `tor` | `tor` + `privoxy` | restart the circuit |
| `proxy` | `curl` + `PROXY_LIST` | next proxy that answers |
| `exec` | your `EGRESS_CMD` | your command runs again |
| `none` | — | no rotation |

### WARP

```bash
curl -fsSL https://pkg.cloudflareclient.com/pubkey.gpg \
  | sudo gpg --yes --dearmor -o /usr/share/keyrings/cloudflare-warp-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/cloudflare-warp-archive-keyring.gpg] \
https://pkg.cloudflareclient.com/ $(. /etc/os-release; echo $VERSION_CODENAME) main" \
  | sudo tee /etc/apt/sources.list.d/cloudflare-client.list
sudo apt-get update && sudo apt-get install -y cloudflare-warp
sudo systemctl enable --now warp-svc
```

OP starts `warp-svc` if it is down, registers the client, switches it to proxy mode
on `WARP_PORT`, connects, and refuses the route unless the proxy reports `warp=on`.
Both CLI syntaxes are handled (`mode proxy` / `proxy port` and the older
`set-mode` / `set-proxy-port`). Rotation is a fresh registration, so each rotation
costs one — that is what the loop guard is for.

`PROXY_LIST` is a URL or a local file, any format: every `ip:port` in it is a
candidate. Free lists are unreliable; WARP is faster and steadier.

`exec` is the escape hatch for anything else — WireGuard, an SSH SOCKS tunnel,
Mullvad, a cloud function:

```bash
EGRESS=exec
EGRESS_CMD='mullvad relay set location de && mullvad connect && echo http://127.0.0.1:8080'
```

It prints the HTTP proxy URL it created, or nothing when the host route changed.

## What a rotation checks

1. The route answers the trace endpoint, and the exit IP differs from the old one
   (IPv6 compared by /64).
2. A TLS handshake with `opencode.ai` succeeds through that same route. An exit
   that answers the trace but fails here is the one that kills the chat a minute
   later with `SSL handshake failed ... net_error -202`.
3. A throwaway session (`run --standalone --archived`) gets an answer and is then
   deleted. A limit answer means the IP is burned; the next one is tried.

Only then does OP resume: `opencode run --auto -c continue`, then the TUI reopens
with `--continue`.

## Loop guard

`ROTATE_GATE_MAX` rotations (default 5) per `ROTATE_GATE_WINDOW` (default 30 min).
Past that OP stops instead of rotating forever. A limit that follows the account
rather than the IP cannot be fixed by another IP, and OP reports it rather than
spinning.

## Config

`~/.config/op/config`, `KEY=VALUE`, one per line, no quotes. Same keys as the
defaults at the top of `start`; see `config.example`. Anything can also be set in
the environment. State and logs live in `~/.local/state/op/` (`op.log` has one
line per rotation, resume, and detected limit).

Useful keys:

```
EGRESS=auto
OC=opencode
PROXY_LIST=
ROTATE_TRIES=8
ROTATE_GATE_MAX=5
PROBE=1
RESUME_MSG=continue
LIMIT_PAT='free-limit|free usage|usage limit|rate limit|429'
ASK=0
```

`LIMIT_PAT` is matched against new lines in
`~/.local/share/opencode/log/*.log`. Only bytes written after the session starts
are read, so an old limit line in a shared log cannot trigger a rotation.

## Requirements

bash 4.3+, curl, opencode on `PATH`. `jq` not needed. Nothing else.

## License

MIT.
