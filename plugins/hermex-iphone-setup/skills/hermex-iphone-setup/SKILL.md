---
name: hermex-iphone-setup
description: "Connect the Hermex iPhone app to a local Hermes dashboard."
version: 1.0.0
license: MIT
author: "the maintainers"
---
# Hermex (iPhone) → local Hermes dashboard

Hermex (native iOS client, `hermexapp.com`) connects **straight to the Hermes
dashboard** on port 9119 — NOT the gateway's api_server (8642) and NOT the
gateway Webapp listener (the ephemeral 55492). It authenticates with
`dashboard.basic_auth` (username + password) and upgrades its live-chat socket
via `POST /auth/password-login` → session cookies → `POST /api/auth/ws-ticket`
→ `/api/ws?ticket=`.

## Read the app's screen FIRST — it is the spec

The Connect screen names the connection mode and the exact fields you must
fill. Read it before any inventory.

- **"Cloudflare Tunnel" selected** → the app wants an HTTPS hostname + a password.
- **"Private network"** (Tailscale, NetBird…) → an address on the private net.
- **"Same Wi-Fi"** → a bare LAN `host:port` over plain HTTP.

The **Password** field is the dashboard's `dashboard.basic_auth` password. The
docs also describe a follow-up username+password prompt once Hermex recognizes
the dashboard, so ship BOTH the username and the password.

## Do the inventory, but do not let it sprawl

The task text names things you are not being asked about. Do the inventory it
asks for — then DROP the adjacent thread. If the goal is "the app on my phone
talks to my agent", auditing every route that account has ever had is not
part of the deliverable. Answer the goal asked.

Find the app's real contract (which port, which listener, which auth), set up
the route, and report. Do not get pulled into unrelated infrastructure.

## The app-mode instruction is part of the answer

The Connect screen's mode selector must MATCH the URL you hand over, or the
app dials the wrong transport and reports "can't reach the server":

| URL you give | Mode the user picks |
|---|---|
| `https://<tunnel-host>` | **Cloudflare Tunnel** |
| `http://100.x.x.x:<port>` (tailnet IP) | **Private network** |
| `http://<lan-ip>:<port>` | **Same Wi-Fi** |

Getting this wrong is the #1 cause of "it can't reach the server" even when
the URL is correct. Say the mode in the same sentence as the URL.

## Pick the route BEFORE editing config

`dashboard.public_url` and the app's connection mode both depend on which
route you chose, and the route can change mid-task (the user may prefer
Tailscale after seeing the Cloudflare setup). Decide the route first, then
write `public_url` once.

A quick tunnel is a legitimate *interim* answer — get the user working today —
but say plainly that the hostname changes on every restart and that the URL
already saved in the app will break. Pair it with the permanent route
immediately so the swap is a known next step, not a surprise.

## Shape that works

1. Dashboard bound to `127.0.0.1:9119` only.
2. A route (Cloudflare Tunnel / Tailscale Serve) terminates HTTPS in front of it.
3. `dashboard.public_url` = the **exact** scheme+host the phone dials. Without it
   the dashboard answers `400 Invalid Host header` (the Host-header guard).
4. `dashboard.basic_auth.username` + `password_hash` (scrypt) + `secret` (≥32 B).
   Setting a non-loopback `public_url` engages the auth gate even on a loopback
   bind — Hermes then fails closed if no provider is registered.

## Password login request shape

`POST /auth/password-login` needs **`provider`** in the body — omitting it
returns `422 Field required: provider`, NOT `401`:

```json
{"provider":"basic","username":"<user>","password":"<pw>"}
```

Success → `200 {"ok":true,"next":"/"}` + `__Host-hermes_session_at`,
`__Host-hermes_session_rt`, `__Host-hermes_session_provider` cookies.
Wrong password → `401 {"detail":"Invalid credentials"}`.

## Hashing + secret (dashboard's own Python)

```bash
cd <hermes-home>/hermes-agent
./venv/bin/python -c "from plugins.dashboard_auth.basic import hash_password; print(hash_password('<pw>'))"
openssl rand -base64 32   # dashboard.basic_auth.secret
```

Store only `password_hash`; leave `password: ''` — a plaintext `password` key
sits in config.yaml in the clear.

## Verifying the whole chain

```bash
curl -s --fail http://127.0.0.1:9119/api/status        # auth_required + auth_providers
curl -s --fail https://<phone-url>/api/status
```
Both must report `"auth_required": true` and `"basic"` in `auth_providers`.
Then confirm login + WS: POST `/auth/password-login` (remember `provider`),
then `POST /api/auth/ws-ticket`, then `GET /api/ws?ticket=…` → `101 Switching Protocols`.

## Cloudflare gotchas (cost real time)

- **Quick tunnels change hostname on every restart.** `dashboard.public_url` and
  the URL saved in Hermex's iOS Keychain both break. Use a **named** tunnel.
- A quick tunnel **inherits `~/.cloudflared/config.yml`**, whose ingress
  catch-all is `http_status:404` → every request 404s. Run it with an isolated
  config file (`--config /path/to/isolated.yml`) or `--url` only.
- `cloudflared tunnel login` writes `~/.cloudflared/cert.pem`; if one exists it
  refuses ("would overwrite"). Move it aside first — back it up, it holds the
  API token for existing tunnels.
- The cert's token is scoped to ONE zone. `cloudflared tunnel route dns` fails
  with `Authentication error` (code 10000) when that zone is no longer on
  Cloudflare (e.g. the domain moved to another registrar). Check with
  `GET /zones?name=<domain>` — an empty `result` means the zone left Cloudflare.
- A CF account with **no zones** can still hold tunnels; it just cannot create
  DNS records. `GET /zones` returning `[]` while
  `GET /accounts/<id>/cfd_tunnel` lists tunnels is the signature.

### Create a named tunnel WITHOUT the browser (when `tunnel login` keeps expiring)

`cloudflared tunnel login` waits ~9 minutes for browser authorization, then
fails with `Failed to fetch resource` — and repeating it burns another 9-minute
window each time. When the zone scope is the only thing blocking you, the API
can do everything except DNS:

```bash
# POST /accounts/<acct>/cfd_tunnel  {name, tunnel_secret}
#   tunnel_secret = base64 of 32 random bytes; "" returns
#   "Tunnel secret can't be empty" (code 1011).
#   The CREATE response carries credentials_file {AccountTag,TunnelID,
#   TunnelName,TunnelSecret} — capture it IN THAT SAME CALL.
#   A later GET /cfd_tunnel/<id> does NOT return credentials_file, so if you
#   drop it, delete the tunnel and create a fresh one (cleanup is
#   DELETE /accounts/<acct>/cfd_tunnel/<id>).

# Write the credentials to ~/.cloudflared/<tunnel-id>.json, mode 400.

# Set the remote ingress config (source becomes "cloudflare"):
# PUT /accounts/<acct>/cfd_tunnel/<id>/configurations
#   {"config":{"ingress":[{"hostname":"<host>","service":"http://localhost:9119"},
#                         {"service":"http_status:404"}],"warp-routing":{}}}

# Run it — cloudflared pulls the remote config and logs
# "Updated to new configuration" + "Registered tunnel connection".
```

Verify with `GET /accounts/<acct>/cfd_tunnel/<id>`: `status: healthy` and a
non-zero `connections` count. This gets you a running named tunnel; only the
DNS CNAME still needs zone write access.

## State the credentials plainly in the final reply

The user WILL ask again. Put the Server URL, username, and password in the
reply as separate labelled lines — not buried in a sentence, not in a table
with other columns. Then say where the password lives: `config.yaml` holds
only the scrypt hash, so the plaintext exists exactly once, in the reply you
just sent.

Also give the app-mode instruction next to the URL, because picking the wrong
connection mode on the Connect screen is the #1 cause of "it can't reach the
server".

## Tunnel + dashboard must both be supervised

Neither survives a logout by default. Two launchd agents (`ai.hermes.dashboard`
and `ai.hermes.cloudflared`) with `RunAtLoad` + `KeepAlive`. Install them
yourself — never configure auto-start unprompted.

- The agent must carry the same flags the working instance used, or it binds
  the wrong address. For the tailnet-IP route that means
  `dashboard --isolated --host <ts-ip> --port 9120 --no-open --skip-build`.
- `--skip-build` reuses the existing `hermes_cli/web_dist` — skip the npm
  build when the dist is already there, it is the difference between a
  10-second start and a multi-minute one.
- Derive the tailnet IP at install time (`tailscale status --json` →
  `Self.TailscaleIPs[0]`) and write it into `dashboard.public_url` in the same
  script. Hardcoding it breaks the moment the machine re-registers.
- `launchctl bootout` the old label before `bootstrap` the new one — a
  re-install over an existing label fails silently and leaves the old agent
  running.

**Writing an installer script that contains `pkill`:** the Hermes terminal
guard blocks any command whose text mentions stopping a gateway-style process,
even when the target is `cloudflared` and the script is only being written.
Have the script stop the tunnel with a narrow pattern
(`pkill -f "cloudflared tunnel --config <path>"`) and let the USER run the
script themselves — which the no-unprompted-auto-start rule wants anyway.

## Same-Wi-Fi fallback

`--host 0.0.0.0` + `http://<machine-ip>:9119`. The password then crosses the
network **unencrypted** — only with explicit confirmation, on a trusted network.

## Tailscale without the Serve feature (works with zero admin toggles)

`tailscale serve` needs TWO tailnet-admin approvals — Serve enabled, and HTTPS
certificates enabled (DNS settings). `tailscale cert <host>` fails with
"your Tailscale account does not support getting TLS certs" until the second
is on. `tailscale serve --bg 9119` then HANGS (it waits on the admin toggle) —
it does not fail fast, so wrap it in a timeout or run it in the background.

**Bypass both gates:** bind the dashboard straight to the tailnet IP.

```bash
TS=/Applications/Tailscale.app/Contents/MacOS/Tailscale
TS_IP=$($TS status --json | python3 -c 'import sys,json; print(json.load(sys.stdin)["Self"]["TailscaleIPs"][0])')
hermes dashboard --isolated --host $TS_IP --port 9120 --no-open
```

- The CLI is NOT on PATH — use `/Applications/Tailscale.app/Contents/MacOS/Tailscale`.
- `--isolated` is REQUIRED: a non-loopback bind otherwise collides with the
  host backend already on 9119 (exit 78). `--isolated` gives it its own server.
- Set `dashboard.public_url` to `http://<TS_IP>:9120` — the exact origin the
  phone dials. Non-loopback => the auth gate engages (good).
- Over plain HTTP the session cookies are BARE (`hermes_session_at`, no
  `__Host-`/`__Secure-` prefix). `_resolved_name` picks the prefix from
  `use_https`, so don't expect `__Host-` cookies on a tailnet HTTP origin.
- Bind stays scoped to the tailnet interface: `lsof` shows
  `TCP 100.x.x.x:9120`, and the LAN IP refuses the connection. Verify that —
  it proves the surface is tailnet-only, not LAN-exposed.
- The tailnet IP is stable for the machine's Tailscale identity, so unlike a
  quick tunnel this URL survives restarts.

Verify the phone side: `tailscale ping <phone-ip>` from the host, and check
`tailscale status --json` shows the phone `Online: true`.

## Related

- Hermes' own docs recommend Tailscale Serve (private, tailnet-only) over
  Cloudflare Tunnel for the open internet, since password sign-in is not meant
  for public exposure.
- If the user wants Cloudflare, the Hermex UI's "Cloudflare Tunnel" mode adds
  `CF-Access-Client-Id` / `CF-Access-Client-Secret` header fields for a
  Cloudflare Access service token. Leave both empty when Access is off.
