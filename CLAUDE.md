# IPRSV1-WIN

Windows Server 2022 host for the IPRSV1 project. Runs the CMSV6 GPS/video
platform and a `frps` (fatedier/frp) reverse-proxy server.

## Connecting to the server

- Access is SSH only, on port **2299**. RDP (3389) is open but not needed for
  any of this repo's work.
- Credentials come from repository variable/secrets `SERVER`, `LOGIN`,
  `PASSWORD` — available in a runner session as env vars `$SERVER`, `$LOGIN`,
  `$PASSWORD`. Auth is password-only, there are no SSH keys.
- Connect with:
  ```bash
  sshpass -p "$PASSWORD" ssh -p 2299 -o StrictHostKeyChecking=accept-new "$LOGIN@$SERVER"
  ```
- The default shell is `cmd`. For anything beyond a single simple command, run
  PowerShell. Do NOT pipe a multi-line script to `powershell -Command -` on
  stdin: it silently stops at the first `foreach`/`if`/`try` block that spans
  more than one line — everything after that block never runs, with no error
  and exit code 0 (verified: a 4-line script with a multi-line `foreach {}`
  printed only the first line). Use `-EncodedCommand` with base64-encoded
  UTF-16LE instead — verified to run multi-line blocks correctly:
  ```bash
  SCRIPT='Write-Output "line1"
  foreach ($i in 1..2) { Write-Output "inside-$i" }
  Write-Output "line-after-block"'
  ENC=$(printf '%s' "$SCRIPT" | iconv -f UTF-8 -t UTF-16LE | base64 -w0)
  sshpass -p "$PASSWORD" ssh -p 2299 "$LOGIN@$SERVER" "powershell -NoProfile -NonInteractive -EncodedCommand $ENC"
  ```
  (If a script truly has no multi-line blocks — every `{ }` on one line — the
  stdin form works too, but `-EncodedCommand` is the safe default.)
- A process started interactively over SSH is killed when the SSH session
  closes (OpenSSH ties the process to the session's job object). Anything that
  needs to keep running independently must go through Windows Task Scheduler
  (see `frps` below), not a backgrounded SSH command.

## What's on the machine

- **CMSV6** — the video/GPS platform (`C:\Program Files\CMSServerV6`):
  services `GPSDaemon`, `GPSGatewaySvr`, `GPSMediaSvr`, `GPSLoginSvr`,
  `GPSUserSvr`, `GPSDownSvr`, `GPSStorageSvr`, `GPSDataProcSvr`,
  `GPSGeocodeSvr`, `GPSFtpd`, web UI on `gpstomcat6` (port 8080), MySQL on
  `GPSMysqld` (127.0.0.1:3311). Do not stop/restart these services, touch
  their files, or change the `CMSv6-*` firewall rules.
- **frps** — reverse-proxy server, `C:\Users\Administrator\Desktop\6e9`
  (`frps.exe`, `frps.ini`, `frps.log`).
- **nginx** — reverse proxy in front of CMSV6's web UI, `C:\nginx` (nginx
  1.30.4). Terminates HTTP on port 80 for `scanvision.online` and
  `www.scanvision.online`; a 443 block also exists but HTTPS isn't reachable
  from outside the machine (see below). **win-acme** — ACME client,
  `C:\win-acme` (win-acme 2.2.9.1701), issues and renews the Let's Encrypt
  certificate nginx uses.

## frps

- Config: `frps.ini` next to the binary, INI format only (this build does not
  understand TOML/YAML). A copy of the vendor-shipped config is kept as
  `frps.ini.vendor-backup`.
- `bind_port = 7000` (frpc clients connect here), `vhost_http_port = 9966`,
  `vhost_https_port = 9967`. Firewall rules `frps-7000-tcp` and
  `frps-9966-9967-tcp` open exactly these ports (point rules by port, same
  style as the existing `CMSv6-*` rules).
- Dashboard binds to **`0.0.0.0:7500`** and is currently reachable from the
  internet — TEMPORARY, opened 2026-07-30 for testing at Максим's request via
  firewall rule `frps-7500-tcp-temp` (the `-temp` suffix marks it for removal
  after testing; not one of the permanent `frps-*` rules). The dashboard
  itself is plain HTTP, no TLS — basic auth (`admin` + `dashboard_pwd`) goes
  over the wire in cleartext while this rule is open. Don't reuse the
  dashboard password anywhere else while it's exposed this way.
  The normal, permanent way to reach the dashboard is still an SSH tunnel —
  use it whenever the temporary firewall rule isn't (or shouldn't be) open:
  ```bash
  sshpass -p "$PASSWORD" ssh -p 2299 -L 7500:127.0.0.1:7500 "$LOGIN@$SERVER"
  ```
  then open `http://127.0.0.1:7500`, user `admin`.
  **To close it back up** once testing is done: delete the firewall rule
  (`Remove-NetFirewallRule -DisplayName "frps-7500-tcp-temp"`), set
  `dashboard_addr = 127.0.0.1` back in `frps.ini`, then restart the `frps`
  scheduled task (`Stop-ScheduledTask` → `Start-ScheduledTask`).
- `token` and the dashboard password (`dashboard_pwd`) are generated values
  that live only in `frps.ini` on the server — not in this repo, not in PRs,
  not in chat or logs. Anyone configuring an `frpc` client reads them by SSH.
  Gotcha carried over from the vendor config: this frps build does **not**
  recognize the key `dashboard_passwd` — use `dashboard_pwd`, or the dashboard
  silently keeps the default password.
- Runs continuously via Windows Task Scheduler task **`frps`**: trigger "at
  system startup", principal `SYSTEM`, working directory
  `C:\Users\Administrator\Desktop\6e9`, restarts on failure. This is
  intentional — see the note above about SSH-attached processes dying with
  the session; only a scheduled task survives independently and across
  reboots.
- `log_file` in `frps.ini` must use forward slashes
  (`C:/Users/Administrator/Desktop/6e9/frps.log`) — a backslash path breaks
  the vendor logger's config parsing (`\U` in `Users` is read as an invalid
  escape sequence) and frps exits immediately without writing a log line.
- `subdomain_host = scanvision.online` is set in `frps.ini` (added 2026-07-30,
  IPRSV1-7). The zone's NS is `ns1.reg.ru`/`ns2.reg.ru`; apex `scanvision.online`
  and `www.scanvision.online` both resolve to the address in `SERVER`. A
  wildcard `*` A record in that zone is a hard requirement for the
  `subdomain`-proxy mechanism to work — frps hands an `frpc` client a hostname
  of the form `<name>.scanvision.online`, and without the wildcard record that
  hostname resolves nowhere. As of 2026-07-30 the wildcard record does **not**
  exist in the zone (confirmed by querying an arbitrary third-level name —
  NXDOMAIN); registrar (reg.ru) credentials aren't held in this repo, so
  someone with reg.ru access needs to add an `A` record, name `*`, value the
  `SERVER` address, in the `scanvision.online` zone. A backup of the
  pre-change config is kept as `frps.ini.bak-2026-07-30` next to `frps.ini`.
- **CMSV6 client linkage.** The vendor connects an `frpc` client (installed by
  the operator, package `CMSV6_WIN_*.exe`, into
  `...\Program Files (x86)\CMSV6\plugin\config\libconfig_<FactoryType>\`) to
  this frps server via a file named `frpcSet.xml` in that same directory:
  ```xml
  <FRPC version="1.0.0">
    <ServerIp>www.scanvision.online</ServerIp>
    <ServerPort>7000</ServerPort>
    <TokenForCon>{значение token из frps.ini}</TokenForCon>
    <ViewPort>9966</ViewPort>
    <isOpenFunc>1</isOpenFunc>
  </FRPC>
  ```
  `ServerPort` is `bind_port`, `ViewPort` is `vhost_http_port` (9966).
  `TokenForCon` is the `token` value from `frps.ini` on this server — read it
  over SSH when configuring a client, never write the actual value here or
  anywhere else outside `frps.ini`. This file and the CMSV6 client that reads
  it live on the operator's machine, not on this server: there is no `D:` drive
  here, no `Program Files (x86)\CMSV6`, and no `libconfig_*`/`frpcSet.xml`
  anywhere on disk — the server only runs the CMSV6 *server* component
  (`C:\Program Files\CMSServerV6`, see above).
- Restarting frps is done **only** through the `frps` scheduled task
  (`Stop-ScheduledTask -TaskName 'frps'`, wait for the `frps.exe` process to
  disappear, then `Start-ScheduledTask -TaskName 'frps'`). Running
  `frps.exe -c frps.ini` directly from an SSH session starts a second,
  competing instance that fights the first for port 7000 and dies the moment
  the SSH session closes — always go through the scheduled task.

## Web access via domain (nginx + win-acme)

CMSV6's web UI (`gpstomcat6`, `127.0.0.1:8080`, no TLS of its own) is reachable
directly by domain name — `http://scanvision.online/` and
`http://www.scanvision.online/` — via an nginx reverse proxy, instead of
requiring the server's IP and port 8080. Port 8080 itself stays open
externally as a fallback entry point; it is not closed by this setup.

- **nginx** — `C:\nginx` (nginx 1.30.4, downloaded from nginx.org; the
  vendor-stated 1.28.x had already rolled to legacy by install time, so
  current stable was used instead). Config at `C:\nginx\conf\nginx.conf`: two
  `server` blocks for `scanvision.online www.scanvision.online`.
  - `listen 80` — serves `/.well-known/acme-challenge/` from `C:/nginx/html`
    (webroot for ACME HTTP-01 validation) and proxies everything else to
    `http://127.0.0.1:8080`.
  - `listen 443 ssl` — same proxy target, `ssl_certificate
    C:/nginx/ssl/scanvision.online-chain.pem`, `ssl_certificate_key
    C:/nginx/ssl/scanvision.online-key.pem`, `ssl_protocols TLSv1.2 TLSv1.3`.
  - Both blocks set `Host`/`X-Real-IP`/`X-Forwarded-For`/`X-Forwarded-Proto`
    and `Upgrade`/`Connection` (via a `map $http_upgrade $connection_upgrade`)
    for CMSV6's websockets, `proxy_buffering off`, 3600s timeouts.
    `client_max_body_size 1024m` is set once at `http` level (inherited by
    both server blocks, not repeated per block).
  - No `Strict-Transport-Security` header and no redirect from 80 to 443 —
    deliberate: HSTS would force browsers to refuse the plain-HTTP entry point,
    and 80 is meant to keep working as a normal, first-class way in (not just
    an ACME-validation path), per Максим's decision.
  - Runs via Windows Task Scheduler task **`nginx`** (same pattern as `frps`:
    trigger at system startup, principal `SYSTEM`, RunLevel Highest, working
    directory `C:\nginx`, restart on failure 3× at 1-minute intervals). Firewall
    rules `nginx-80-tcp` and `nginx-443-tcp` (by port, same style as `CMSv6-*`
    and `frps-*`) open the two ports.
  - **Reload gotcha:** `nginx.exe -s reload` only works when run by the same
    Windows account that started the nginx master process — it signals a
    named Windows event (`Global\ngx_reload_<pid>`) scoped to that account.
    Since the scheduled task runs nginx as `SYSTEM`, running `nginx.exe -s
    reload` from an interactive SSH session (which runs as the login account,
    e.g. `Administrator`) fails with `OpenEvent(...) failed (5: Access is
    denied)` — verified. To apply a config change manually over SSH, restart
    the scheduled task instead: `Stop-ScheduledTask -TaskName nginx` →
    (wait for both `nginx.exe` processes to exit) → `Start-ScheduledTask
    -TaskName nginx`. This gotcha does **not** affect win-acme's automatic
    renewal: its own scheduled task also runs as `SYSTEM`, so the post-install
    script it calls (`C:\nginx\reload-nginx.bat`: `cd /d C:\nginx` +
    `nginx.exe -s reload`) runs in the same account as nginx and succeeds.
  - **Working-directory gotcha:** `nginx.exe -t` (or `-s reload`) run from an
    SSH session's default directory fails on relative paths — nginx resolves
    `conf`/`logs` relative to its own working directory, not its exe location
    — e.g. `nginx: [alert] could not open error log file: CreateFile()
    "logs/error.log" failed` and `CreateFile()
    "C:\Users\Administrator/conf/nginx.conf" failed`. Always run either from
    `C:\nginx` (`cd C:\nginx` first) or with explicit `-p`/`-c`:
    `nginx.exe -p C:\nginx -c conf\nginx.conf -t`.
- **win-acme** — `C:\win-acme` (win-acme 2.2.9.1701, `wacs.exe`). Issued the
  certificate with:
  ```
  wacs.exe --source manual --host scanvision.online,www.scanvision.online ^
    --friendlyname scanvision.online ^
    --validation filesystem --webroot C:\nginx\html ^
    --store pemfiles --pemfilespath C:\nginx\ssl ^
    --installation script --script C:\nginx\reload-nginx.bat ^
    --accepttos --emailaddress admin@scanvision.online
  ```
  Certificate (issuer `CN=YR1, O=Let's Encrypt, C=US`) covers both names,
  valid 2026-07-30 → 2026-10-28. Files in `C:\nginx\ssl`:
  `scanvision.online-chain.pem` (full chain, used as `ssl_certificate`),
  `-key.pem` (private key, used as `ssl_certificate_key`), plus `-crt.pem`
  (leaf only) and `-chain-only.pem` (CA chain only) which nginx doesn't use.
  Renewal runs automatically via scheduled task **`win-acme renew
  (acme-v02.api.letsencrypt.org)`** (`SYSTEM`, next due 2026-09-23) — no
  manual action needed under normal operation. `--renew --force` ignores the
  due date but does **not** bypass win-acme's own cache — a cached order/pfx
  from the last issuance is reused for about a day afterwards ("Using cache
  for scanvision.online... Order 1/1 (Main): handle from cache" in
  `C:\ProgramData\win-acme\<baseuri>\Log\`), so running `--force` shortly
  after an issuance just re-applies the same certificate (store + install
  hooks run, no new ACME order or http-01 validation happens). To force a
  genuinely new certificate, add `--nocache` — and that's specifically what
  the weekly Let's Encrypt duplicate-certificate limit (5 per name set per
  rolling week) applies to; a bare `--force` right after issuance doesn't
  count against it. What actually happened 2026-07-30: a `--renew --force`
  run at 20:49 (about 30 minutes after the initial issuance at ~20:22) hit
  the cache — pem files in `C:\nginx\ssl` were rewritten and nginx reloaded
  cleanly via a temporary SYSTEM-context scheduled task (interactive SSH
  runs as `Administrator`, which would hit the reload gotcha above), but no
  new certificate was issued and the end-to-end ACME/http-01 cycle itself was
  only exercised once, by the original issuance.
- **Proxy behavior verified from real traffic**, not just synthetic checks:
  `C:\nginx\logs\access.log` shows real clients successfully logging in
  through the proxy and CMSV6's main-interface websocket
  (`/ws/webSocket/index/1`, `/ws/webSocket/down/1`) returning `101` with no
  `400`/`502` from the proxy path. That socket builds its URL from
  `this.location.host`/`this.location.protocol` in the client JS, so it
  follows whatever scheme the page was loaded with. Live video specifically:
  `access.log` (five days, 2026-07-31 → 2026-08-03, ~7k real requests) shows
  the player's pages and scripts loading fine through the proxy — 200 on
  `/808gps/ttxvideo/ttxvideo-h5.html`, `/808gps/open/player/video-replay.html`,
  `ttxplayer-h5.js`, `cmsv6player.min.js`, `ttxvideoapi.js` — but the actual
  video stream itself never goes through nginx: the only websocket upgrades
  in that whole window are `/ws/webSocket/index/1` and
  `/ws/webSocket/down/1` (the main interface socket); no `flv`/`m3u8`/`hls`/
  `mp4` request and no other websocket path appears at all. So the video
  stream bypasses the proxy entirely and goes straight to CMSV6's media port,
  not through `scanvision.online`. That's consistent with the risk flagged in
  the original spec (video URLs coming from the API as absolute
  addresses/ports rather than relative to the page) — it isn't specifically
  a mixed-content/HTTPS problem, since HTTPS isn't externally reachable
  anyway (see below). `http://scanvision.online/` and direct
  `http://<адрес из SERVER>:8080/` are both unaffected either way, since
  neither depends on the video stream routing through nginx.

### Known limitation: HTTPS (443) not reachable from outside

The 443 listener, firewall rule, and certificate are all in place and correct
on this machine, but external clients cannot complete a TLS handshake to
`scanvision.online:443` — the connection hangs until timeout. This is **not**
something wrong on this server; it was confirmed by direct measurement, not
guessed:

- a plain TCP connection to port 443 from outside completes normally;
- a plain (non-TLS) HTTP request sent to port 443 from outside gets a real
  `400 Bad Request` back from nginx — so data does flow both ways on that
  port, just not when the payload is a TLS handshake;
- a TLS handshake from outside to the *other* externally-open ports on this
  same machine — 9966, 9967, 7500, 8080 — completes instantly; TLS traffic as
  a class is not being stripped on the path;
- from the machine itself, TLS to port 443 (via `127.0.0.1` and via its own
  public address) completes in ~50ms with a valid response.

Conclusion: something on the network path between the internet and this
server — outside Windows, outside this machine entirely — is specifically
dropping the combination "port 443 + TLS handshake". Where exactly is not
distinguishable from the vantage points available (can't tell if the inbound
ClientHello or the outbound ServerHello is the one being lost).

This machine's hosting is **not Aliyun**, despite that being assumed when the
task was first written: hostname `VDSWIN2K22`, network adapter is a generic
Red Hat VirtIO device, and the Aliyun/AWS/Azure/GCP cloud metadata endpoints
are all unreachable from the server. There is no cloud "security group" to
open here. Максим added an allow rule for 443 in the hosting provider's own
panel on 2026-07-30 — it did not change the observed behavior.

Nothing further was done about this in-task — it needs a decision from
Максим. The nginx 443 config, certificate, and firewall rule are all left in
place and working; if the network-path filtering is ever lifted, HTTPS should
start working immediately with no further changes, and just needs an external
check to confirm. Two possible workarounds exist but were deliberately not
implemented here (both need Максим's decision, and change the setup rather
than just fix a bug):

1. Serve TLS on one of the ports that measurably isn't filtered (9966, 9967,
   7500, or 8080) instead of 443 — works, but the domain would need the port
   in the URL.
2. Put an external reverse proxy/CDN in front of the domain that terminates
   TLS itself and talks to this server over plain port 80.

## Secrets

Never write actual values of `LOGIN`, `PASSWORD`, `SERVER`, `token`, or
`dashboard_pwd` into this repo, PR descriptions, commit messages, or command
logs — only their names and how they're used.
