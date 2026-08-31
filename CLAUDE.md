# IPRSV1-WIN

Windows Server 2022 host (`VDSWIN2K22`, Timeweb VDS) for the IPRSV1 project.
Runs the CMSV6 GPS/video platform, an `nginx` TLS front-end and an `frps`
(fatedier/frp) reverse-proxy server used to reach in-vehicle recorders.

**Everything below marked "verified DD.MM.YYYY" was measured on the machine or
from outside on that date.** Facts without such a mark come from earlier work
and were not re-measured. The last full sweep was **31.08.2026**.

## Connecting to the server

- Access is SSH. **Port 22** (verified 31.08.2026 — `sshd_config` has no
  explicit `Port` directive, and 22 is the only sshd listener; the port 2299
  documented here previously is gone).
- Two ways in:
  - **SSH key** (preferred): an `ssh-ed25519` key with comment `reg@thedevs.ru`
    is installed in `C:\ProgramData\ssh\administrators_authorized_keys`
    (`sshd_config` has `Match Group administrators` →
    `__PROGRAMDATA__/ssh/administrators_authorized_keys`, so per-user
    `.ssh/authorized_keys` under the admin profile is ignored). Verified
    31.08.2026:
    ```bash
    ssh -i ~/.ssh/id_ed25519 Administrator@$SERVER
    ```
  - **SSH key from the k8s AI runners** (added 31.08.2026): a second
    `ssh-ed25519` key with comment `ai-runner@ai-runners` is installed in the
    same `administrators_authorized_keys` (fingerprint
    `SHA256:VNCdf2dCCZB3FzRgjJAB7NudjfCy8jqyO4HJxVWdlhU`). Its private half is
    stored **only in the Kubernetes secret `iprsv1-win-ssh`** in namespace
    `ai-runners` on the 192.168.88.248 cluster — keys `id_ed25519` and
    `id_ed25519.pub`. Role/RoleBinding `iprsv1-win-ssh-reader` lets every
    service account in that namespace `get` **this one secret and no other**,
    so any pod there can use the key without a deployment change.
    Each pod's `~/.ssh/config` has an alias plus a
    `Match host iprsv1 exec "/home/user/.ssh/iprsv1-key.sh"` hook that pulls
    the key out of the secret on first use and `chmod 600`s it, so from inside
    a pod it is just:
    ```bash
    ssh iprsv1 "hostname"
    ```
    To do it by hand from anywhere with cluster access:
    ```bash
    microk8s kubectl -n ai-runners get secret iprsv1-win-ssh \
      -o jsonpath='{.data.id_ed25519}' | base64 -d > ~/.ssh/id_ed25519
    chmod 600 ~/.ssh/id_ed25519
    ```
    Backups of the pre-change key file are on the server as
    `administrators_authorized_keys.bak-20260831` and `.bak-20260831b`.
  - **Password**, from repository variables/secrets `SERVER`, `LOGIN`,
    `PASSWORD` (env `$SERVER`, `$LOGIN`, `$PASSWORD` in a runner session):
    ```bash
    sshpass -p "$PASSWORD" ssh -o StrictHostKeyChecking=accept-new "$LOGIN@$SERVER"
    ```
- RDP (3389) is open but not needed for any of this repo's work.
- The default shell is `cmd`. For anything beyond a single simple command, run
  PowerShell. Do NOT pipe a multi-line script to `powershell -Command -` on
  stdin: it silently stops at the first `foreach`/`if`/`try` block that spans
  more than one line — everything after that block never runs, with no error
  and exit code 0. Use `-EncodedCommand` with base64-encoded UTF-16LE instead:
  ```bash
  ENC=$(iconv -f UTF-8 -t UTF-16LE script.ps1 | base64 -w0)
  ssh -i ~/.ssh/id_ed25519 Administrator@$SERVER \
    "powershell -NoProfile -NonInteractive -EncodedCommand $ENC"
  ```
  Put `$ProgressPreference = 'SilentlyContinue'` on the first line of any
  script that calls `Get-NetFirewallRule`, `Get-ScheduledTask` and friends —
  otherwise PowerShell streams CLIXML progress records into stdout and buries
  the real output (verified 31.08.2026).
- A process started interactively over SSH is killed when the SSH session
  closes (OpenSSH ties the process to the session's job object). Anything that
  needs to keep running independently must go through Windows Task Scheduler,
  not a backgrounded SSH command.

## What's on the machine

Verified 31.08.2026 unless noted.

- **CMSV6** — the video/GPS platform. The `gpstomcat6` service starts from
  `C:\Program Files\CMSServerV6\7.37.2_20260710\tomcat\bin\gpstomcat6.exe`
  (service `PathName`). The `CMSServerV6old\` tree documented here earlier is
  no longer the live one — **always read the service `PathName` before
  touching any CMSV6 file**, reinstalls have moved this path twice already.
  Services: `GPSDaemon`, `GPSGatewaySvr`, `GPSMediaSvr`, `GPSLoginSvr`,
  `GPSUserSvr`, `GPSDownSvr`, `GPSStorageSvr`, `GPSDataProcSvr`,
  `GPSGeocodeSvr`, `GPSFtpd`, web UI on `gpstomcat6`
  (port 80), MySQL on `GPSMysqld` (127.0.0.1:3311). Listening ports observed:
  2121 + 14147 (FTPd), 6601–6613, 6617, 6625 (loopback, tomcat shutdown),
  6630–6635, 9999. Do not stop/restart these services, touch their files, or
  change the `CMSv6-*`/`GPS *` firewall rules — the one documented exception
  is `gpstomcat6`, restarted to apply a `server.xml` port change.
- **nginx** — `C:\nginx` (nginx 1.30.4). Listens on **443 only**. See
  "Web entry points" below.
- **frps** — reverse-proxy server, now at **`C:\frps`** (moved off
  `C:\Users\Administrator\Desktop\6e9`, which no longer exists). Version
  **0.17.0**. See "frps" below. ✅ **Up, dashboard closed, as of 31.08.2026** —
  see "Current state: frps is UP, dashboard closed" under "frps" for the
  measurements.
- **win-acme** — ACME client, now at **`C:\wacs`** (`C:\win-acme` no longer
  exists), win-acme 2.2.9.1701. Its scheduled task
  `win-acme renew (acme-v02.api.letsencrypt.org)` is **enabled** and healthy
  (last run 31.08.2026, result 0). It holds exactly **one** renewal:
  `[Manual] test.thedevs.ru`, due 28.09.2026. It no longer issues or renews
  anything for `scanvision.online` — that name uses the purchased GlobalSign
  certificate (below).
- **Zabbix Agent** — service `Zabbix Agent`, running, listening on 10050. Not
  installed by this project's work; left alone.

### Scheduled tasks (autostart)

| Task | Action | Notes |
|---|---|---|
| `nginx_run` | `C:\nginx\run_nginx.bat` | at startup, SYSTEM. The `.bat` is idempotent — it checks `tasklist` for a running `nginx.exe` and only then `cd /d C:\nginx` + `start "" nginx.exe`, so it cannot raise a second master. |
| `frps` | `C:\frps\run-frps.bat` | at startup, SYSTEM. The `.bat` is `cd /d C:\frps` + `frps.exe -c C:\frps\frps.ini` in the foreground — when frps exits, the task ends with it. |
| `win-acme renew (…)` | `C:\wacs\wacs.exe` | vendor task, renews `test.thedevs.ru` only. |

A second, duplicate `nginx` task existed briefly on 14.08.2026 (with its own
`run-nginx.bat`) and was **deleted** — two startup triggers would have raced
for port 443. Do not recreate it; `nginx_run` is the only one.

### Firewall (inbound, enabled)

Verified 31.08.2026. The `frps-*` per-port rules documented here previously no
longer exist; the frps ports are now covered by one CMSV6-style rule:

- `GPS services TCP 2` — TCP **7000, 9966, 9967, 20021, 22** (7500 removed
  31.08.2026, IPRSV1-16 — the frps dashboard on that port is no longer
  exposed, see `## frps` below)
- `nginx 80` — TCP 80 (the listener behind it is CMSV6's tomcat, not nginx)
- `nginx 443` — TCP 443
- `OpenSSH Server (sshd)` — TCP 22
- the vendor's `GPSNginx`/`GPS *` rules — protocol Any, left untouched

## Web entry points (80 / 443)

Both work from the internet (verified 31.08.2026):

| URL | Result | Served by |
|---|---|---|
| `http://scanvision.online/` | 200 | CMSV6 tomcat directly on port 80 |
| `https://scanvision.online/` | 200, valid chain | nginx 443 → `127.0.0.1:80` |

**HTTPS from outside now works.** The "Known limitation: HTTPS (443) not
reachable from outside" that dominated this file until 04.08.2026 is
**resolved** — a full TLS handshake plus a 200 response from an external RU
vantage point took 0.59 s on 31.08.2026. Keep this in mind when reading old
specs in `specs/`: their 443 caveats are historical.

### CMSV6 tomcat on port 80

The connector lives in `tomcat\conf\server.xml` of the live CMSV6 tree
(currently `C:\Program Files\CMSServerV6\7.37.2_20260710\tomcat`, read the
service `PathName` first). It is:

```xml
<Connector connectionTimeout="20000" port="80"
           protocol="org.apache.coyote.http11.Http11Nio2Protocol"
           redirectPort="8443" ... />
```

- 🔴 **Never add `address="127.0.0.1"` to this connector.** The CMSV6 desktop
  client talks to the platform port directly (the address is handed to it by
  `GPSLoginSvr`), and with a loopback-only bind the client hangs in `SynSent`.
- Restarting `gpstomcat6` costs **~3.5 minutes** of platform downtime. That is
  normal for CMSV6, not a hang — do not "fix" it by killing the process.
- **Reinstall gotcha:** any CMSV6 update or reinstall overwrites `server.xml`
  from the vendor package and puts the connector back on 8080. After every
  such update, set port 80 again and restart `gpstomcat6` — otherwise *both*
  entry points break: port 80 stops answering and 443 starts returning 502,
  since nginx proxies to `http://127.0.0.1:80`.
- Port 8080 is not in use any more; `http://<SERVER>/` is the fallback entry
  point.

### nginx

`C:\nginx\conf\nginx.conf` (verified 31.08.2026) has **two** `server` blocks,
both `listen 443 ssl`, both `proxy_pass http://127.0.0.1:80`:

1. `test.thedevs.ru` — Let's Encrypt cert from `C:/nginx/conf/le/`
   (`test.thedevs.ru-chain.pem` / `-key.pem`), renewed by the win-acme task.
2. `scanvision.online www.scanvision.online` — purchased GlobalSign cert from
   `C:/nginx/conf/scanvision/` (`scanvision-chain.crt` / `scanvision.key`),
   plus OCSP stapling (`ssl_stapling on`, `ssl_trusted_certificate
   scanvision-trusted.crt`, `resolver 8.8.8.8 8.8.4.4`).

Both blocks set `Host`, `X-Real-IP`, `X-Forwarded-For`, `X-Forwarded-Proto`.
There is **no** `listen 80` block (CMSV6 owns 80), no ACME webroot location,
no HSTS and no redirect to 443 — port 80 is a first-class way in, deliberately.

The cert/key paths documented here before (`C:\nginx\ssl\scanvision.online-gs-*.pem`)
are gone — `C:\nginx\ssl` does not exist any more. Config backups sit next to
the live file: `nginx.conf.bak`, `nginx.conf.bak2`, `nginx.conf.bak-20260814`.

- 🔴 **Missing websocket proxying — see "Known issue" below.**
- **Reload gotcha:** `nginx.exe -s reload` only works when run by the account
  that started the master process. The scheduled task runs nginx as `SYSTEM`,
  so a reload from an interactive SSH session (running as `Administrator`)
  fails with `OpenEvent(...) failed (5: Access is denied)`. To apply a config
  change over SSH: `taskkill /F /IM nginx.exe`, then
  `schtasks /run /tn nginx_run`.
- **Working-directory gotcha:** `nginx.exe -t` from an SSH session's default
  directory fails on relative paths (`could not open error log file:
  CreateFile() "logs/error.log" failed`) — nginx resolves `conf`/`logs`
  against its working directory, not its exe location. Run from `C:\nginx` or
  with explicit `-p`/`-c`: `nginx.exe -p C:\nginx -c conf\nginx.conf -t`.
- **Stale-master gotcha:** a master process detached from the task's job
  object survives a task stop/start cycle and keeps listening in parallel with
  the fresh instance, so requests get served randomly by either config/cert
  (observed 04.08.2026). After any restart, check `Get-Process nginx`
  `StartTime` and that `Get-NetTCPConnection -State Listen | Where LocalPort
  -eq 443` shows only the new PIDs.

### Certificates

- **Purchased GlobalSign** — in use on 443 for `scanvision.online`. Verified
  from outside 31.08.2026: issuer `GlobalSign GCC R46 DV TLS CA 2025`, leaf
  `CN=www.scanvision.online`, SAN = `www.scanvision.online`,
  `scanvision.online`, `autodiscover.`, `mail.`, `owa.`, valid
  **03.08.2026 → 18.02.2027**.
  🔴 **There is no auto-renewal for it. Replace it manually before 18.02.2027
  or HTTPS stops working with no warning** — nothing in this repo or on the
  server monitors that date.
  🔴 **It is not a wildcard**: device subdomains (`<devid>.scanvision.online`,
  see frps below) are *not* covered, so a browser warns on them. Reaching a
  recorder over plain HTTP on :9966 is the intended path.
  The original kit from the registrar is on the server in
  `C:\Users\Administrator\Desktop\srt` (`certificate.crt` / `certificate.key` /
  `certificate_ca.crt`; the `key.txt` in that folder is unrelated, don't use
  it). That desktop copy of the private key is unencrypted and not ACL-locked —
  locking it down or removing it is Максим's call, still not done.
  When replacing the cert, assemble the chain by hand the same way: leaf, then
  intermediate, no root, no BOM.
- **Let's Encrypt** — only for `test.thedevs.ru` now (renewal due 28.09.2026).
  Renewal for `scanvision.online` was cancelled on 04.08.2026 and its old
  `.pem` files are gone along with `C:\nginx\ssl`.

## DNS

Zone `scanvision.online`, NS `ns1.reg.ru` / `ns2.reg.ru`. Verified via public
DoH (`dns.google`) on 31.08.2026 — do not check this from a machine behind the
office VPN, its resolver caches stale answers far longer:

- `scanvision.online` → `201.34.132.26`
- `www.scanvision.online` → `201.34.132.26`
- **`*.scanvision.online` → `201.34.132.26`** — the wildcard A record that the
  frps `subdomain` mechanism requires **now exists** (an arbitrary
  third-level name resolves; this file previously said it was missing).

## frps

Config `C:\frps\frps.ini`, INI format only (this build does not understand
TOML/YAML). Verified 31.08.2026:

```ini
bind_port = 7000            # frpc clients connect here
vhost_http_port = 9966      # device subdomains are served here
vhost_https_port = 9967
subdomain_host = scanvision.online
tcp_mux = true
dashboard_addr = 127.0.0.1
dashboard_port = 7500
dashboard_user = admin
# frps 0.17.0: dashboard_pwd only, dashboard_passwd is silently ignored
dashboard_pwd = ...
token = ...
allow_ports = 65535
max_ports_per_client = 1
log_file = C:/frps/frps-run.log
log_way = file
log_level = info
log_max_days = 30
```

- ⚠️ **`dashboard_passwd` is not a valid key for frps 0.17.0 — it only knows
  `dashboard_pwd`.** This is a permanent trap, not a one-time bug: the
  vendor's own example configs use `dashboard_passwd`, and this build's parser
  silently ignores an unknown key instead of erroring. That exact mistake is
  what left the dashboard on the default `admin`/`admin` for weeks — if the
  wrong key ever comes back (a future rewrite, a copy from a vendor example),
  the dashboard reverts to `admin`/`admin` with no error at all. Always check
  the key actually present is `dashboard_pwd`.
- **Closed since 31.08.2026 (IPRSV1-16):** `dashboard_addr = 127.0.0.1` and
  firewall rule `GPS services TCP 2` no longer includes 7500 — the dashboard
  is unreachable from the internet. Reach it only from the machine itself or
  through an SSH tunnel: `ssh -i ~/.ssh/id_ed25519 -N -L 7500:127.0.0.1:7500
  Administrator@$SERVER`. `admin`/`admin` now gets `401`.
- **What this build's dashboard actually exposes** — checked against upstream
  `v0.17.0` source, not assumed: read-only routes only (`/api/serverinfo`,
  per-protocol proxy lists, `/api/proxy/traffic/:name`, static UI), no route
  returns the `token`. `/api/serverinfo` gives version, `subdomain_host`,
  port/timeout config and traffic counters; the proxy-list routes give the
  registered proxy **names, which are recorder Device IDs**, plus traffic.
  So the real exposure while it sat open was version, `subdomain_host`, the
  list of connected Device IDs and traffic volumes — not the token.
- `log_file` must use forward slashes — a backslash path breaks the vendor
  logger's config parsing (`\U` in `Users` read as an invalid escape) and frps
  exits immediately without writing a log line.
- The vendor's own `frps.log` (04.08.2026) is a leftover from the shipped
  package — its errors are not ours. The live log is `frps-run.log`, rotated
  daily (`frps-run.YYYY-MM-DD.log`).
- Restart **only** through the scheduled task (`schtasks /run /tn frps`, or
  `Stop-ScheduledTask`/`Start-ScheduledTask`). Running `frps.exe -c frps.ini`
  from an SSH session starts a second instance that fights the first for 7000
  and dies with the session.
- Config backups: `frps.ini.bak-20260814`, `frps.ini.bak-before-dash`,
  `frps.ini.bak-before-dash0`, `frps.ini.bak-before-iprsv1-16` (31.08.2026 —
  holds the pre-rotation `token` and the old `dashboard_passwd`/
  `dashboard_addr = 0.0.0.0`; restore it over `frps.ini` and restart the
  `frps` task to roll back this change).

### Token rotation (31.08.2026, IPRSV1-16)

`token` and `dashboard_pwd` were rotated in the same edit that closed the
dashboard, in one restart — the old token had sat reachable in plaintext over
HTTP for weeks, so it's treated as burned regardless of whether anyone
actually read it.

- the new value lives only in `C:\frps\frps.ini` — read it over SSH, never
  copy it anywhere else: `Select-String -Path C:\frps\frps.ini -Pattern
  '^token'`;
- the previous value survives only in `C:\frps\frps.ini.bak-before-iprsv1-16`
  (rollback copy, see above).

**Hard cutover** — 0.17.0 has exactly one token, no transition period. Every
recorder and every operator's CMSV6 client still holding the old token fails
frps authorization the instant the new config went live. Field checklist
(a human does this — there is no access from this server to recorder or
operator machines):

1. read the new token over SSH as above;
2. on every recorder and every operator machine, update **both**
   `C:\Program Files (x86)\CMSV6\config\frpc.ini` and
   `...\CMSV6\plugin\config\libconfig_<FactoryType>\frpcSet.xml` (element
   `<TokenForCon>`);
3. restart the CMSV6 client on each machine;
4. confirm through the SSH-tunnel dashboard that proxies (`/api/proxy/http`)
   reappear, named by Device ID.

Until step 2 is done everywhere, **no recorder will show up in the
dashboard** — that is the expected result of the rotation, not a bug in this
change. Rollback: restore `C:\frps\frps.ini.bak-before-iprsv1-16` over
`frps.ini` and restart the `frps` scheduled task.

### Restored hardening: allow_ports / max_ports_per_client / log_max_days

`allow_ports = 65535`, `max_ports_per_client = 1` and `log_max_days = 30` were
set by the 31.07.2026 hardening (IPRSV1-9) and silently lost when the
14.08.2026 port rework rewrote `frps.ini` from scratch. Restored 31.08.2026
(IPRSV1-16) — **whoever edits `frps.ini` next must carry these three keys
forward too**; this file is hand-edited each time, not generated from a
template that would preserve them automatically.

- 0.17.0's only access control is the single shared `token` — there are no
  per-client ACLs or per-proxy allowlists, so `allow_ports`/
  `max_ports_per_client` are the one limit available on what a valid token
  can do;
- CMSV6 is unaffected by `max_ports_per_client = 1` — checked against
  upstream source (`server/proxy.go`), not assumed: `usedPortsNum` is
  incremented only for `TcpProxyConf`/`UdpProxyConf`. A recorder's
  `subdomain`-based proxy is an `HttpProxyConf`, which never increments it, so
  `server/control.go`'s `RegisterProxy` comparison against
  `max_ports_per_client` never trips for CMSV6 traffic.

### ✅ Current state: frps is UP, dashboard closed (as of 31.08.2026)

Measured after this task's (IPRSV1-16) restart:

- `frps.exe` running, exactly one process, brought up by
  `Start-ScheduledTask -TaskName 'frps'` at 31.08.2026 16:46;
- fresh `frps-run.log` lines: `frps tcp listen on 0.0.0.0:7000`, `http service
  listen on 0.0.0.0:9966`, `https service listen on 0.0.0.0:9967`,
  **`Dashboard listen on 127.0.0.1:7500`**, `Start frps success`;
- externally: 7000/9966/9967 accept connections, 7500 times out;
  `http://scanvision.online/` and `https://scanvision.online/` both `200`;
- `Get-NetTCPConnection -State Listen -LocalPort 7500` shows only
  `127.0.0.1`.

**Unexplained fact found by this task's pre-check, before its own restart:**
the scheduled task was already `Running` with a live `frps.exe`
(`LastRunTime` 31.08.2026 16:42:42) — contradicting the "down since
17.08.2026 20:16" state this file recorded until now: the last line in
`frps-run.log` before this task was **17.08.2026 20:16:05**, and the task's
`LastTaskResult` was **267014** (`0x41306`, "last run terminated by user").
Something restarted it between the last sweep and this check; **the cause is
not established here** and is out of this task's scope — frps' startup/restart
reliability belongs to IPRSV1-14. This task restarted it anyway, through its
own normal `Stop-`/`Start-ScheduledTask` procedure, to apply the new config
regardless of what was already running.

### How a recorder is reached

- `frpc` runs **on the recorder itself** (linux/arm), not on this server. It
  dials `frps` on 7000 and registers a subdomain equal to its Device ID.
  Confirmed in the log on 17.08.2026: client login from `81.9.28.145`, version
  0.17.0, os linux, arch arm → `http proxy listen for host
  [900000400273.scanvision.online]`.
- Entry point is the frps vhost port: `http://<devid>.scanvision.online:9966/`
  returns the recorder's own web page (~19 KB). An unregistered ID gets a 404
  from frps. There are **no** device subdomains on port 80 — that is CMSV6.
- The CMSV6 desktop client keeps its own frp settings locally on the
  operator's machine, in `C:\Program Files (x86)\CMSV6\config\frpc.ini` and
  `...\CMSV6\plugin\config\libconfig_<FactoryType>\frpcSet.xml`:
  ```xml
  <FRPC version="1.0.0">
    <ServerIp>www.scanvision.online</ServerIp>
    <ServerPort>7000</ServerPort>
    <TokenForCon>{token from frps.ini}</TokenForCon>
    <ViewPort>9966</ViewPort>
    <isOpenFunc>1</isOpenFunc>
  </FRPC>
  ```
  `ServerPort` = `bind_port`, `ViewPort` = `vhost_http_port`. Read the token
  over SSH when configuring a client; never write its value anywhere outside
  `frps.ini`. None of this exists on the server — it only runs the CMSV6
  *server* component.
- **Since the 31.08.2026 token rotation (IPRSV1-16):** any recorder or
  operator CMSV6 client still holding the pre-rotation token fails frps
  authorization and registers no proxy — see `### Token rotation` above for
  the field update and rollback procedure.

### ⏰ Open issue: "Accessing FRPS timeout is not responding!"

The CMSV6 desktop client shows this message when its FRPS feature is used.
**Cause not established.** What was actually observed: on click the plugin
connects only to 6601/6603/6607 and to the CMSV6 web port — it never touches
7000 or 9966 — and no Windows client ever appears in the frps login log. Next
step is packet-level capture on the operator's machine, not more guessing on
the server.

## 🔴 Known issue: websockets do not survive the 443 proxy

Reproduced 31.08.2026 from outside, same request, two ports:

| Request | Result |
|---|---|
| `https://scanvision.online/ws/webSocket/index/1` with `Upgrade: websocket` | **404** |
| `http://scanvision.online/ws/webSocket/index/1` with `Upgrade: websocket` | **101 Switching Protocols** |

Cause (direct, from the config): the current `nginx.conf` sets no
`proxy_http_version 1.1` and no `Upgrade`/`Connection` headers, so nginx
downgrades the request to a plain HTTP/1.0 proxy call and tomcat answers 404.
An earlier revision of this config had a `map $http_upgrade $connection_upgrade`
block plus `proxy_buffering off` and 3600 s timeouts; the current file (rebuilt
14.08.2026) does not. Also missing from `http {}`: `client_max_body_size`
(default 1 m, so uploads over 1 MB through 443 get 413).

Effect: CMSV6's main-interface sockets (`/ws/webSocket/index/1`,
`/ws/webSocket/down/1`) work over `http://` and fail over `https://`. Not yet
fixed — the fix is four lines in each `location /`, but it needs an nginx
restart on a live platform, so it is Максим's call.

Separately, and unchanged: the **video stream never goes through nginx at all**.
Five days of `access.log` (31.07–03.08.2026, ~7k real requests) show the
player's pages and scripts loading through the proxy (`ttxvideo-h5.html`,
`video-replay.html`, `ttxplayer-h5.js`, `cmsv6player.min.js`) but no
`flv`/`m3u8`/`hls`/`mp4` request and no websocket path other than the two
above — the stream goes straight to CMSV6's media port, because the API hands
out absolute addresses.

## History: 14.08.2026 port rework, rolled back the same day

Done and reverted on the same day, kept here so the backup files make sense:

- **Attempted:** nginx in front on both 80 and 443, CMSV6 tomcat moved to
  8081, frps vhost moved to 9970.
- **Reverted the same evening at Максим's request** (verified 14.08.2026
  20:50): nginx covers **443 only**, CMSV6 is back on **80** directly, frps
  vhost back on **9966/9967**.
- Left behind by that day: `*.bak-20260814` backups next to `frps.ini` and
  `nginx.conf`, the deleted duplicate `nginx` scheduled task (see above), the
  move of frps from `Desktop\6e9` to `C:\frps`, and `dashboard_addr = 0.0.0.0`
  (re-opened at Максим's request).

## Secrets

Never write actual values of `LOGIN`, `PASSWORD`, `SERVER`, the frps `token`,
or the dashboard password into this repo, PR descriptions, commit messages, or
command logs — only their names and how they're used.
