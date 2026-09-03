# IPRSV1-WIN

**This repo now describes TWO servers of the IPRSV1 project.** Everything
below, up to "Сервер 2 — 200.165.238.242" at the end of this file, describes
server 1 — `201.34.132.26` (`VDSWIN2K22`, Timeweb VDS) — unless a line says
otherwise. Server 2, added 03.09.2026 (IPRSV1-18), has its own section at the
end; the two are separate machines with a shared SSH key
(`ai-runner@ai-runners`, same public half on both, by design — see that
section) and, since 03.09.2026 (IPRSV1-18, second spec), a **shared `frps`
token/dashboard password** — see "Сервер 2" → frps.ini for the price of that
(paired rotation). No other shared configuration.

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
  **0.17.0**. See "frps" below. ✅ **Up, self-healing, dashboard closed, as of
  31.08.2026** — see "Current state: frps is UP, self-healing, dashboard
  closed" under "frps" for the measurements.
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
| `frps` | `C:\frps\run-frps.bat` | at startup + every 5 min (repeats forever), SYSTEM. The `.bat` is idempotent (`tasklist` check, same pattern as `nginx_run`'s), the task has no `ExecutionTimeLimit`, `MultipleInstances IgnoreNew`, and restarts on failure (1 min × 3). See "frps" section below for the 17.08.2026 outage and how it's watched now. |
| `frps-watchdog` | `powershell.exe -NoProfile -NonInteractive -ExecutionPolicy Bypass -File C:\frps\frps-watchdog.ps1` | every 5 min (offset from `frps`'s own trigger), SYSTEM, `ExecutionTimeLimit PT5M`. No-op when `frps.exe` is alive; otherwise unconditionally re-triggers `frps` (`Stop-ScheduledTask` first, only if the `frps` task's own state is still `Running`). See "frps" section below. |
| `win-acme renew (…)` | `C:\wacs\wacs.exe` | vendor task, renews `test.thedevs.ru` only. |

A second, duplicate `nginx` task existed briefly on 14.08.2026 (with its own
`run-nginx.bat`) and was **deleted** — two startup triggers would have raced
for port 443. Do not recreate it; `nginx_run` is the only one.

### Firewall (inbound, enabled)

Verified 31.08.2026, re-measured same day for IPRSV1-13 (before and after
creating the two rules below — 100 enabled inbound allow rules before, 102
after, no other rule touched). The `frps-*` per-port rules documented here
previously no longer exist. The `CMSv6-*`-named rules seen on 30.07.2026
(`CMSv6-6601-6612-tcp`/`-udp`, `CMSv6-6631-6635-tcp`,
`CMSv6-2121-2122-6617-2162-8080-tcp`) are also gone — CMSV6 has been
reinstalled since, and the vendor now ships the same coverage under new names:

- `GPS services TCP` — TCP **80, 443, 6617, 6631-6635, 2122-2162, 6601-6612**
- `GPS services UDP` — UDP **6601-6612**
- `GPS services TCP 2` — TCP **7000, 9966, 9967, 20021, 22** (7500 removed
  31.08.2026, IPRSV1-16 — the frps dashboard on that port is no longer
  exposed, see `## frps` below)
- `nginx 80` — TCP 80 (the listener behind it is CMSV6's tomcat, not nginx)
- `nginx 443` — TCP 443
- `OpenSSH Server (sshd)` — TCP 22
- `OpenSSH SSH Server Preview (sshd)` — TCP 22, a second enabled rule for the
  same port
- the vendor's `GPSNginx`/`GPS *` rules (`GPSLoginSvr`, `GPSGatewaySvr`,
  `GPSMediaSvr`, `GPSUserSvr`, `GPSDownSvr`, `GPSServerControl`,
  `GPSStorageSvr`, `GPSTomcat`, `GPSFtpd`, `GPSGeocodeSvr`, `GPSRedisd`) —
  each is a *pair* of rules, one TCP and one UDP, both with `LocalPort Any`
  (confirmed via `Get-NetFirewallPortFilter`; not "protocol Any") — scoped by
  program rather than port, left untouched; their actual port coverage
  cannot be read from the firewall alone.
- `iprsv1-13-ports-tcp` (IPRSV1-13, created 31.08.2026) — TCP **80, 88, 443,
  2121-2162, 6601-6612, 6617, 6630-6635, 8080, 8088, 16601, 16603-16605,
  16607-16609, 16611, 20000-21000, 30000-31000**
- `iprsv1-13-ports-udp` (IPRSV1-13, created 31.08.2026) — UDP **6602-6612,
  20000-21000, 30000-31000**
- `iprsv1-17-ports-tcp` (IPRSV1-17, created 31.08.2026) — TCP **16604, 16605**,
  `-Profile Any -RemoteAddress Any -Program Any`, for the two nginx media-proxy
  listeners above. Note: **16605 and 16604 were already covered** by
  `iprsv1-13-ports-tcp`'s `16603-16605` range (confirmed — both ports answered
  from outside before this rule existed), so this rule is a deliberate
  task-scoped duplicate for documentation clarity, same pattern as the
  existing named rules, not a functional gap-fill.

**IPRSV1-13 (31.08.2026): decision by Максим — open the full requested list,
overlap with the dynamic port range accepted.** He was shown the risk first
(list entries `16601`, `16603-16605`, `16607-16609`, `16611`, `20000-21000`,
`30000-31000` fall inside this machine's 9000-64999 ephemeral range — see
"Dynamic port range" below — so a future reboot could land a listening
`svchost`/`gps*` process on one of them under an already-open port) and chose
to accept it rather than withhold those entries, verbatim in the task chat:
«открывай все что я попросил неважно сидит там кто-то или нет». The two
rules above were created exactly as specified — full normalized list,
`-Profile Any -RemoteAddress Any -Program Any`. The dynamic port range itself
was **not** changed (still 9000-64999) — narrowing it back to the 49152
default was a separate option that was not chosen.

Coverage measured **before** this task (list vs. the rules above and a live
listener snapshot, both now superseded by the two new rules): open — TCP 80,
443, 2122-2162, 6601-6612, 6617, 6631-6635, and the single port 20021 (no
listener there at the time); UDP 6601-6612 (i.e. the list's 6602-6612). Not
covered by any concrete rule and no listener present either — TCP 88, 8080,
8088, all of 16601/16603-16605/16607-16609/16611, and
20000-21000/30000-31000 besides 20021; UDP 20000-21000 and 30000-31000 in
full. Two edge cases: TCP 2121 had a listener (CMSV6 FTPd) but only the
ambiguous `GPSFtpd` Any-program rule, no concrete port rule; TCP 6630 had a
listener but sat just outside `GPS services TCP`'s `6631-6635`.

Verified **after** the change: external TCP connect from the runner succeeds
for the list's ports that already had a listener (2121, 6605, 6617, 6630,
6632). Ports with a rule but no listener (88, 8080, 8088, the 16xxx block,
and most of 20000-21000/30000-31000) don't answer from outside — expected,
not a failure; for those the rule's existence is the only evidence. UDP is
not checked externally at all — no responder on the other end means no way
to tell "open" from "closed" that way.

### Dynamic port range

Verified 31.08.2026 (IPRSV1-13): `netsh int ipv4 show dynamicport tcp` and
`netsh int ipv4 show dynamicport udp` both report **Start Port 9000, Number
of Ports 56000** — i.e. ports **9000-64999** are Windows' ephemeral range on
this machine for both protocols, not the platform default of 49152-65535.
IPRSV1-13 did **not** change this range — Максим was shown the overlap with
six of the requested port-list entries and chose to open them anyway rather
than narrow the range (see "Firewall" above).

The actual risk of an inbound-allow rule landing inside this range is **not**
that it exposes an outbound connection's own ephemeral source port — an
inbound allow rule doesn't do that: an outbound socket is bound by its own
5-tuple, and a stranger's unsolicited SYN to that local port simply doesn't
match any connection and is dropped. The real risk is an ephemerally
**listening** socket coming up inside the range later, e.g. after a reboot,
landing on a port this range now permits inbound. Measured 31.08.2026: TCP
49664-49670 is currently held by `lsass`/`wininit`/`services`/`spoolsv`/
`svchost`, and roughly a dozen UDP ports in 33019-61300 by
`gps*`/`nginx`/`svchost` — none of that is fixed or guaranteed to reuse the
same ports next time. Any future inbound firewall rule touching 9000-64999
needs the same check IPRSV1-13 did: measure the range and the current
listeners first, and treat anything that later starts listening inside an
already-open range as an accepted, not a new, exposure.

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

`C:\nginx\conf\nginx.conf` (verified 31.08.2026) has **four** `server`
blocks: two `listen 443 ssl` (both `proxy_pass http://127.0.0.1:80`), plus
the two media-proxy listeners on 16605/16604 added by IPRSV1-17 (see
"Media-under-HTTPS listeners" below). The two 443 blocks:

1. `test.thedevs.ru` — Let's Encrypt cert from `C:/nginx/conf/le/`
   (`test.thedevs.ru-chain.pem` / `-key.pem`), renewed by the win-acme task.
2. `scanvision.online www.scanvision.online` — purchased GlobalSign cert from
   `C:/nginx/conf/scanvision/` (`scanvision-chain.crt` / `scanvision.key`),
   plus OCSP stapling (`ssl_stapling on`, `ssl_trusted_certificate
   scanvision-trusted.crt`, `resolver 8.8.8.8 8.8.4.4`).

Both blocks set `Host`, `X-Real-IP`, `X-Forwarded-For`, `X-Forwarded-Proto`,
and (added 31.08.2026, IPRSV1-15) six more directives in the same `location /`:
`proxy_http_version 1.1;`, `proxy_set_header Upgrade $http_upgrade;`,
`proxy_set_header Connection $connection_upgrade;`, `proxy_buffering off;`,
`proxy_read_timeout 3600s;`, `proxy_send_timeout 3600s;`. There is **no**
`listen 80` block (CMSV6 owns 80), no ACME webroot location, no HSTS and no
redirect to 443 — port 80 is a first-class way in, deliberately.

`proxy_buffering off` and the 3600 s timeouts sit on the shared `location /`,
not on a dedicated `location /ws/` — deliberately: CMSV6 has two known ws
paths (`/ws/webSocket/index/1`, `/ws/webSocket/down/1`) but that list comes
from log analysis and isn't guaranteed complete, and a path outside a
dedicated `/ws/` location would silently fall back to the default 60 s
timeout and drop under idle. The trade-off is that ordinary HTTP traffic on
this host now also runs unbuffered with long timeouts — accepted because
traffic here is small (~7k requests/5 days, see `access.log` analysis below).

The cert/key paths documented here before (`C:\nginx\ssl\scanvision.online-gs-*.pem`)
are gone — `C:\nginx\ssl` does not exist any more. Config backups sit next to
the live file: `nginx.conf.bak`, `nginx.conf.bak2`, `nginx.conf.bak-20260814`,
`nginx.conf.bak-20260831` (IPRSV1-15, before adding the six ws directives
above), `nginx.conf.bak-20260831b` (IPRSV1-17, before adding the two
media-port listeners below).

At the `http {}` level: `client_max_body_size 512m;` (added 31.08.2026,
IPRSV1-15 — see "Fixed: websockets…" below for why 512m) and a
`map $http_upgrade $connection_upgrade { default upgrade; '' close; }` used by
all four `location /` blocks (the two 443 blocks above and the two
IPRSV1-17 media-proxy blocks below) for websocket upgrade.

**Media-under-HTTPS listeners (IPRSV1-17, added 31.08.2026).** Two more
`server` blocks, same cert as the `scanvision.online` 443 block, no
`ssl_stapling` (only needed on the public 443 entry point):

- `listen 16605 ssl;` → `proxy_pass http://127.0.0.1:6605;` — CMSV6's
  `GPSLoginSvr` (the login server). The H5 player's video/archive/PTT flow
  starts here: under `https:`, the page GETs
  `https://scanvision.online:16605/3/1?...` (i.e. login-server port + 10000)
  to resolve the media server's address before ever touching a media port.
- `listen 16604 ssl;` → `proxy_pass http://127.0.0.1:6604;` — CMSV6's
  `GPSMediaSvr` (the media server). This is where the actual `wss://` video
  stream (`clientPort` from the `/3/1` response, also +10000) lands; live
  video and archive playback multiplex over this one port — confirmed by
  sweeping `MediaType`/`Type` combinations against the login server, which
  returned the same `clientPort:6604` in every case. **Intercom (PTT) is not
  covered by that sweep** — it uses a second, runtime-assigned port handed
  out only after a successful ptt-login (`B4(...)` in `cmsv6player.min.js`),
  so whether it also lands on 6604 is unmeasured, not confirmed. See "CMSV6
  platform config" below for why a working listener here still wasn't enough
  on its own.
- If DevTools ever shows a PTT `wss://` connection on a port other than
  16604/16605 (the intercom's second, runtime-assigned port, unmeasured — see
  above), add the identical pattern for it: nginx listener at `port+10000`,
  firewall port, and an entry in the `HttpsMapHttpPort` map.

- ✅ **Websocket proxying — fixed 31.08.2026, see "Fixed: websockets now
  survive the 443 proxy" below.**
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

### CMSV6 platform config exposed to clients (IsHttps, server addresses)

The H5 player (`808gps/js/cmsv6player.min.js`) does not build `wss://` from
the page's own `https:` protocol alone — it first GETs
`http(s)://<login-server-host>:<login-server-port[+10000 under https]>/3/1?MediaType=..&Type=..&AVType=..&DevIDNO=..&Channel=..`
and only switches to `wss://` if that response's `IsHttps` field is truthy.
The one-shot check-tool for all of this, from the server itself:

```
GET http://127.0.0.1:6605/3/1?MediaType=1&Type=0&AVType=1&DevIDNO=900000400219&Channel=0
```

Fields that matter in the response: `IsHttps`, `server.clientIp`/`lanip`
(what the player uses as the stream host — falls back to these unless
`location.hostname` already equals one of `clientIp`/`clientIp2`/`clientIp3`/
`lanip`), `server.clientPort` (the media port, mapped through `HttpsMapPort`
or `+10000` by default), `HttpsMapPort` (only present once `IsHttps` is true).

**Where these values live:**

- `server.clientIp`/`lanip`/`clientPort`/`deviceIp`/`devicePort` for the
  media server (`svrid: 3` in the response) come from `1010GPS.server_info`,
  row `ID=3` (`IDNO='M1'`, `Name='Media Server'`). 🔴 This table is **not**
  empty on this machine (12 rows, present since 04.08.2026) — an earlier
  investigation assumed it was; always `SELECT * FROM server_info` before
  touching it, an `INSERT` against an already-populated table would create a
  conflicting/duplicate row. To publish a domain name instead of the IP,
  `UPDATE` the existing row's `IPClient`/`LanIP` columns only —
  `IPDevice`/`PortDevice`/`PortClient` must stay exactly as measured
  (GPS devices, not browsers, connect via `IPDevice`/`PortDevice`, and those
  are not the values being fixed here). Done 31.08.2026 (IPRSV1-17):
  `IPClient`/`LanIP` → `scanvision.online` on `ID=3`; confirmed live only
  after a `GPSLoginSvr` restart — this table is read once at service start,
  not per-request.
- The `HttpsMapHttpPort`→`HttpsMapPort` map is the pre-existing (empty)
  `1010GPS.jt808_system_params` row `id=61` (`data_type`/`data_code` both
  `HttpsMapHttpPort`). Format is `src:dst;src:dst`, matched 1:1 with the
  player's own map parser. Set 31.08.2026 to `6604:16604;6605:16605`
  (identical to the `+10000` default, made explicit so the mapping is visible
  in the DB and extensible for a future intercom port). Also read once at
  service start, and — per the finding below — appears to only surface in the
  `/3/1` response at all once `IsHttps` is true.
- 🔴 **`IsHttps` — the flag itself has no documented handle, and neither
  candidate mechanism found by reading `libmsgproc_clientmedia.dll` /
  `libvehicleinfo_jt808.dll` worked, even after a `GPSLoginSvr` restart
  (approved by Максим, 31.08.2026, task chat IPRSV1-17):**
  1. A `jt808_system_params` row `data_type='Https'`, `data_code='HttpsPort'`,
     `data_value='443'` (same shape as `id=61`/`62`) — tested and rolled back,
     no effect even after restart.
  2. An autodetected `<CMSV6 install>\nginx\conf\nginx.conf` (the vendor's own
     nginx tree, which ships no `nginx.conf` out of the box — only sample
     `nginx-*.conf` files) with one `listen 443 ssl` block — tested and rolled
     back, no effect even after restart. The vendor nginx itself is **not**
     started for this — the file is read as data, not executed.
  Both hypotheses are exhausted; `IsHttps` still reports `0`. **Next step is a
  vendor support request, not further guessing** — do not try more
  `jt808_system_params` rows at random and do not edit the vendor DLLs/JS.
  Until this is resolved, the browser player will keep building `ws://` (not
  `wss://`) under `https:` pages and mixed-content-block itself before ever
  reaching the nginx listeners below, regardless of how well those listeners
  work on their own.
- Both DB changes above (and both `IsHttps` experiments) were confirmed to
  need a `GPSLoginSvr` restart to take effect — this service caches
  `server_info`/`jt808_system_params` at startup rather than reading them
  per-request. `GPSLoginSvr` is a separate, comparatively cheap restart from
  `gpstomcat6` (no ~3.5 min downtime observed) but is still a live-platform
  action — get explicit sign-off before restarting it, same as for any other
  CMSV6 service.
- Pre-change dumps of both touched tables sit next to the nginx config
  backups, not in the repo: `C:\nginx\conf\server_info.bak-20260831.sql`,
  `C:\nginx\conf\jt808_system_params.bak-20260831.sql`
  (`mysqldump`, `1010GPS` DB on `127.0.0.1:3311`; credentials in
  `tomcat\webapps\gpsweb\WEB-INF\classes\config\jdbc.properties` under the
  live CMSV6 tree — not repeated here, see "Access is SSH" above for the
  don't-commit-secrets rule).

## DNS

Zone `scanvision.online`, NS `ns1.reg.ru` / `ns2.reg.ru`. Verified via public
DoH (`dns.google`) on 31.08.2026 — do not check this from a machine behind the
office VPN, its resolver caches stale answers far longer:

- `scanvision.online` → `201.34.132.26`
- `www.scanvision.online` → `201.34.132.26`
- **`*.scanvision.online` → `201.34.132.26`** — the wildcard A record that the
  frps `subdomain` mechanism requires **now exists** (an arbitrary
  third-level name resolves; this file previously said it was missing).

**Zone `scan-vision.ru`** (сервер 2, `200.165.238.242`), NS `reg.ru`. Verified
via public DoH (`dns.google`) 03.09.2026 — records already existed before
IPRSV1-18 touched anything, nothing here was created or changed by this
project:

- `scan-vision.ru` → `200.165.238.242`
- `www.scan-vision.ru` → `200.165.238.242`
- `*.scan-vision.ru` → `200.165.238.242` (wildcard, confirmed with an
  arbitrary third-level probe name) — required by the frps `subdomain`
  mechanism, same role as the `scanvision.online` wildcard above.
- TTL 21600.

## frps

Config `C:\frps\frps.ini`, INI format only (this build does not understand
TOML/YAML). Verified 31.08.2026:

```ini
bind_port = 7000
vhost_http_port = 9966
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

### ✅ Current state: frps is UP, self-healing, dashboard closed (as of 31.08.2026)

Two independent tasks changed this server within the same hour, in this
order — kept straight here because their timestamps otherwise look
contradictory:

1. **16:01, IPRSV1-14** raised frps by hand (`schtasks /run /tn frps`) after
   the 17.08.2026 20:16 outage (cause investigated below) and fixed the
   scheduled task so a stop like that can't repeat unnoticed (mechanism in
   "Как frps теперь присматривается" below).
2. **16:15–16:46, IPRSV1-14** — while reconfiguring the task and running
   diagnostics from several concurrent SSH sessions, one task instance
   (started 16:27:01) ended up stuck in `Running` for **~19 minutes** with a
   live, healthy `frps.exe` behind it: `run-frps.bat`'s last line runs
   `frps.exe` in the foreground, so its wrapper only exits when the process
   does — by design, not a bug — and `IgnoreNew` correctly blocked the
   5-minute trigger's retries at 16:32, 16:37 and 16:42 because an instance
   really was still running. **This is what explains IPRSV1-16's
   "unexplained fact"** ("task already `Running`, `LastRunTime` 16:42:42"
   before it had touched anything) — reproduced directly, 31.08.2026 evening:
   `Get-ScheduledTaskInfo`'s `LastRunTime` advances on every 5-minute trigger
   *evaluation*, including one `IgnoreNew` skips, not only on an instance
   that actually starts. Watched it happen live: process PID/`StartTime`
   unchanged across a trigger boundary, yet `LastRunTime` still advanced in
   step with the grid, from 18:27 to 18:32 (`Get-ScheduledTaskInfo`'s
   `LastRunTime`/`NextRunTime` mirror the minute value into the seconds
   field — e.g. `18:27:27`, `NextRunTime 18:52:52` — that's a display
   artifact, not a real offset: read them as `18:27`, `18:52`. There is no
   tens-of-seconds lag, and `16:42:42` above is likewise just `16:42`). Not a
   restart, phantom or otherwise — confirmed explained, not left
   unestablished.
3. **16:45:53–16:46, IPRSV1-16** rewrote `frps.ini` (dashboard bound to
   `127.0.0.1`, `token`/`dashboard_pwd` rotated) and stopped the still-stuck
   16:27 instance (`Stop-ScheduledTask`, logged as "stopped ... as request by
   user Administrator") to restart the task and apply it. That restart
   produced the `frps.exe` process that then ran undisturbed for ~1.5 hours.

**Auto-recovery test, 31.08.2026 evening (clean, back-to-back — supersedes an
earlier in-session estimate that turned out not to match the event log on
review):** `taskkill /F /IM frps.exe`, twice, each waited out to full
recovery:

- kill 1 at 18:22:04 → new process listening on all four ports at 18:24:05 —
  **121 s**, recovered by `frps-watchdog`: its own 5-minute trigger fired at
  18:24:01, found no live `frps.exe`, and called `schtasks /run /tn frps`
  unconditionally 4 s later (event 110, on-demand run) — there is no
  `RestartOnFailure` event anywhere between the 18:22:04 kill and 18:24:01;
- kill 2 at 18:24:20 → new process listening at 18:27:01 — **161 s**,
  recovered via `frps`'s own 5-minute repeating trigger this time (tagged
  "due to a time trigger condition", event 107) — `frps-watchdog`'s next
  tick hadn't come up yet;
- both times: exactly one `frps.exe` process, all four ports owned by its
  PID, well under the 5-minute target either way.

Two recovery paths are actually verified, not three: `frps-watchdog`
(kill 1) and `frps`'s own repeating trigger (kill 2). `RestartOnFailure`
did not fire in either test: `run-frps.bat`'s `cmd.exe` wrapper does exit
with the killed child's own non-zero code (`2147942401` / `0x80070001`), but
Task Scheduler logs that as a plain "successfully completed" action (event
201), not as a failure that hands off to `RestartOnFailure` — don't rely on
it for this task's specific kill/wrapper combination. What actually closes
the "stuck `Running` with no live process" gap is `frps-watchdog`, and it is
not a passive zombie-cleaner — see "Как frps теперь присматривается" below
for how it really behaves.

**Measured after IPRSV1-16's restart (still true):**

- `frps.exe` running, exactly one process, brought up by
  `Start-ScheduledTask -TaskName 'frps'` at 31.08.2026 16:46;
- fresh `frps-run.log` lines: `frps tcp listen on 0.0.0.0:7000`, `http service
  listen on 0.0.0.0:9966`, `https service listen on 0.0.0.0:9967`,
  **`Dashboard listen on 127.0.0.1:7500`**, `Start frps success`;
- externally: 7000/9966/9967 accept connections; `:7500` times out — closed on
  purpose (IPRSV1-16's firewall-rule and `dashboard_addr` change above), not a
  regression from IPRSV1-14, which never touches `frps.ini` or firewall rules.
  A made-up subdomain on `:9966` still returns **404 from frps** — vhost
  routing works end to end and does not depend on the dashboard, which is
  what actually satisfies IPRSV1-14's own external-reachability criterion now
  that `:7500` is intentionally closed;
- `http://scanvision.online/` and `https://scanvision.online/` both `200`;
- `Get-NetTCPConnection -State Listen -LocalPort 7500` shows only
  `127.0.0.1`;
- no `frpc` client has re-registered since — expected, see "Token rotation"
  above: every recorder's `frpc.ini`/`frpcSet.xml` still holds the
  pre-rotation token until a human updates it.

**17.08.2026 20:16 stop — cause still not established.** Evidence taken
31.08.2026 before any change (saved on the machine as
`C:\frps\frps-task-before-2026-08-31.xml`, the rollback path):

- the `ExecutionTimeLimit = PT72H` hypothesis previously written here is
  **refuted**: the pre-change task XML has **no `<ExecutionTimeLimit>`
  element at all**, not `PT72H`;
- excluded directly by the evidence: an idle-triggered stop
  (`<RunOnlyIfIdle>` was absent from the XML, i.e. false, so the
  `<IdleSettings>` block present alongside it was inert); an OS-level
  crash/power-loss/resource exhaustion around 17.08.2026 20:16 (`System` and
  `Application` logs reach back to 02.08.2026 04:07, fully covering
  17.08 18:00–18.08 02:00, and hold nothing but routine service start/stop
  noise — no `Kernel-Power`, no `Resource-Exhaustion-Detector`, no
  `Application Error`/`Application Hang` for `frps.exe`); a panic inside frps
  itself — the log's last lines are ordinary `Accept new mux stream error:
  broken pipe` warnings, no panic, no shutdown line;
- can't be excluded, because no telemetry existed to check it either way: an
  explicit `schtasks /end`/`Stop-ScheduledTask` call — the one channel that
  would show it, `Microsoft-Windows-TaskScheduler/Operational`, was
  **disabled** (confirmed 31.08.2026) with zero history for 14–18.08.2026,
  now fixed (see below); the task's own
  `<StopIfGoingOnBatteries>true</StopIfGoingOnBatteries>` — implausible on a
  VDS with no battery, but nothing rules it out;
- the `Security` log's own oldest entry is 30.08.2026 20:08 (~20h retention,
  confirming the note further down that it covers under a day) — it never
  reached 17.08.2026, so event 4689 was never checkable either way.

Do not add a cause beyond what's above — none of it is confirmed, only some
of it is excluded.

### Как frps теперь присматривается (since 31.08.2026)

- `C:\frps\run-frps.bat` is idempotent, same pattern as `C:\nginx\run_nginx.bat`:
  a `tasklist` check exits immediately if `frps.exe` is already running,
  otherwise it starts it in the foreground. Old version backed up on the
  machine as `run-frps.bat.bak-2026-08-31`.
- Task `frps`: `<ExecutionTimeLimit>PT0S</ExecutionTimeLimit>` (no limit),
  two triggers — at startup, and every 5 minutes with no end —
  `MultipleInstances IgnoreNew`, `RestartOnFailure` 1 min × 3, principal
  unchanged (`SYSTEM`, `HighestAvailable`). Action unchanged, still
  `C:\frps\run-frps.bat`.
- `Microsoft-Windows-TaskScheduler/Operational` is enabled (20 MB, was
  disabled before 31.08.2026) — next time frps stops, that channel and
  `C:\frps\frps-run.log` are the first two places to look.
- **`frps-watchdog` (added during IPRSV1-14 review round 1, 31.08.2026):** a
  second scheduled task, `C:\frps\frps-watchdog.ps1`
  (`powershell.exe -NoProfile -NonInteractive -ExecutionPolicy Bypass -File
  ...`), SYSTEM/HighestAvailable, `IgnoreNew`, running every 5 minutes offset
  from `frps`'s own trigger, with its own bounded `ExecutionTimeLimit = PT5M`
  (unlike `frps`'s `PT0S`, because this action is always meant to finish in
  seconds). **Not a passive zombie-cleaner — it's a real third recovery
  path, and the one that actually fired in testing:** the script exits
  immediately and touches nothing only if `Get-Process frps` finds a live
  process; otherwise it calls `schtasks /run /tn frps` **unconditionally** —
  `Stop-ScheduledTask -TaskName frps` runs first only as a conditional
  cleanup, when the `frps` task's own `State` is still `Running` (the
  stuck-instance case, ~19 minutes, above). Confirmed live, 31.08.2026
  evening: after `taskkill /F /IM frps.exe` with the `frps` task in plain
  `Ready` state (no stuck instance, so the `Stop-ScheduledTask` branch never
  ran), `frps-watchdog`'s own trigger found no live process and raised
  `frps` unconditionally — that's kill 1 above (121 s). Only the
  `Stop-ScheduledTask` cleanup branch remains untested live — forcing a real
  stuck instance to test it means another live outage.
- **Known gap in `frps-watchdog` itself:** both `Get-Process`/
  `Stop-ScheduledTask` calls swallow errors (`-ErrorAction
  SilentlyContinue`), `schtasks /run`'s own exit code is discarded
  (`Out-Null`), and the script always exits `0` — so
  `Microsoft-Windows-TaskScheduler/Operational` logs the same "success, code
  0" whether a tick did nothing or actually restarted `frps`. Already cost
  real time once: reconstructing which mechanism recovered kill 1 above took
  cross-referencing several event IDs instead of reading it off directly.
  Left as-is on purpose for this task — a log line
  (`C:\frps\frps-watchdog.log`) or `Write-EventLog` call at the point it
  actually acts, plus checking `schtasks /run`'s own exit code, would close
  this; not done here.
- **`frps`'s own `LastTaskResult`/`LastRunTime` are no longer useful health
  signals in normal operation.** With the 5-minute repeating trigger plus
  `IgnoreNew`, every 5-minute tick while `frps.exe` is alive gets refused —
  observed live, 31.08.2026 evening, with `frps.exe` healthy since 18:27:01:
  `LastTaskResult = 2147946720` (`net helpmsg 4320` → "The operator or
  administrator has refused the request", `0x800710E0`). That is now the
  *normal*, healthy reading — not the `267014`/`0x41306` ("terminated by the
  scheduler") that flagged the original 17.08.2026 outage, and it no longer
  means anything is wrong on its own. Also, `Get-ScheduledTaskInfo`'s
  `LastRunTime`/`NextRunTime` display their minute value mirrored into the
  seconds field (e.g. `18:47:47`, `NextRunTime 18:52:52`) — read as
  `18:47`/`18:52`, not literal timestamps to the second. Check liveness with
  `Get-Process frps` and the four ports, not either field.

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

## ✅ Fixed: websockets now survive the 443 proxy (31.08.2026, IPRSV1-15)

**Before**, reproduced 31.08.2026 13:00 UTC from outside, same request, two ports:

| Request | Result |
|---|---|
| `https://scanvision.online/ws/webSocket/index/1` with `Upgrade: websocket` | **404** |
| `http://scanvision.online/ws/webSocket/index/1` with `Upgrade: websocket` | **101 Switching Protocols** |

Cause (direct, from the config): `nginx.conf` set no `proxy_http_version 1.1`
and no `Upgrade`/`Connection` headers, so nginx downgraded the request to a
plain HTTP/1.0 proxy call and tomcat answered 404. **Correction of a claim
this file made before 31.08.2026:** there was no earlier config revision with
`map $http_upgrade $connection_upgrade` — `findstr` over `nginx.conf.bak`,
`.bak2` and `.bak-20260814` on 31.08.2026 found no ws directives in any of
them (while a control search for `proxy_pass` matched all three, so the files
do get read), so the fix below is an addition, not a restore. Also missing
from `http {}`: `client_max_body_size` (default 1 m, so uploads over 1 MB
through 443 got 413).

**Fix applied 31.08.2026 (IPRSV1-15):** backed up the live config to
`nginx.conf.bak-20260831`, then added at the `http {}` level
`map $http_upgrade $connection_upgrade { default upgrade; '' close; }` and
`client_max_body_size 512m;`, and in both `location /` blocks:
`proxy_http_version 1.1;`, `proxy_set_header Upgrade $http_upgrade;`,
`proxy_set_header Connection $connection_upgrade;`, `proxy_buffering off;`,
`proxy_read_timeout 3600s;`, `proxy_send_timeout 3600s;`. Applied via
`taskkill /F /IM nginx.exe` + `schtasks /run /tn nginx_run` (the only way to
apply a config change, see "Reload gotcha" above) — verified no orphaned
master afterwards.

`512m` is deliberate, not arbitrary: CMSV6's own upload ceiling is 500 MB
(`web.xml`'s `<max-file-size>524288000</max-file-size>`,
`struts.properties`'s `struts.multipart.maxFileSize=524288000`, connector
`maxPostSize="-1"`), so nginx's limit sits just above the application's, kept
as the gate.

**After**, verified 31.08.2026 from outside, same request as above:

| Request | Result |
|---|---|
| `https://scanvision.online/ws/webSocket/index/1` | **101**, `Sec-WebSocket-Accept` present |
| `https://scanvision.online/ws/webSocket/down/1` | **101** |
| `https://test.thedevs.ru/ws/webSocket/index/1` | **101** |
| `http://scanvision.online/ws/webSocket/index/1` and `/down/1` | still **101** — port 80 behaviour unchanged |
| `https://scanvision.online/` and `https://test.thedevs.ru/` | **200**, cert chain valid — plain HTTPS unaffected |
| POST 2 000 000 bytes to `https://scanvision.online/` | full body reached the backend (`size_upload=2000000`, no 413); response now byte-identical to the same POST over `http://` |

Effect: CMSV6's main-interface sockets (`/ws/webSocket/index/1`,
`/ws/webSocket/down/1`) now work over both `http://` and `https://`.

The remaining acceptance criterion — the main CMSV6 interface at
`https://scanvision.online/` visibly updating data in real time — is a
by-eye browser check, not something scriptable from here; it is left to
Максим to confirm. The `101`/byte-identical-body checks above cover the
protocol-level behaviour the browser check would rely on.

Separately, and unchanged: the **video stream never goes through nginx at
all** over plain `http://` — the player connects straight to CMSV6's media
port (`ws://<ip>:6604`), because the API hands out absolute addresses; five
days of `access.log` (31.07–03.08.2026, ~7k real requests) show the player's
pages and scripts loading through the proxy (`ttxvideo-h5.html`,
`video-replay.html`, `ttxplayer-h5.js`, `cmsv6player.min.js`) but no
`flv`/`m3u8`/`hls`/`mp4` request and no websocket path other than the two
above. **Under `https:`, as of IPRSV1-17 (31.08.2026), nginx does now have a
listener in that path** (`16604 → 127.0.0.1:6604`, see "Media-under-HTTPS
listeners" above) — but the player still won't use it: CMSV6's own `IsHttps`
flag stays `0`, so under `https:` pages the player keeps building `ws://`
(not `wss://`) and the browser blocks it as mixed content before any nginx
listener is ever reached. See "CMSV6 platform config" above for the
exhausted-hypotheses state of that flag.

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

## Сервер 2 — 200.165.238.242

Развёрнут задачей IPRSV1-18 (03.09.2026) как второй, независимый сервер
проекта — тот же функционал CMSV6+frps, что на сервере 1 выше. Задачей
IPRSV1-18 (вторая спека, 03.09.2026) доведён до состояния, близкого к
близнецу сервера 1: свой домен `scan-vision.ru`, `nginx` + Let's Encrypt
(win-acme), общий с сервером 1 `frps`-токен. **Ничего из раздела ниже не
относится к серверу 1 (`201.34.132.26`) — он не менялся, к нему было только
чтение (значения frps и версии nginx/win-acme).**

### Подключение (verified 03.09.2026)

- SSH, порт 22 (включён вручную человеком через RDP — снаружи у машины
  изначально был открыт только 3389, автоматически включить SSH из пода было
  нечем: `xfreerdp`/`nmap`/`sudo`/`apt` в поде отсутствуют).
- Вход ключом `ai-runner@ai-runners` (та же публичная половина, что на
  сервере 1, из secret `iprsv1-win-ssh`, namespace `ai-runners`) — установлен
  в `C:\ProgramData\ssh\administrators_authorized_keys`, наследование прав
  снято, оставлены `Administrators`+`SYSTEM`. Пароль `Administrator` (в
  описании задачи IPRSV1-18 на доске) после установки ключа для работы не
  нужен.
- Многострочный PowerShell — только через `-EncodedCommand`, как на сервере
  1; первой строкой скриптов, читающих `Get-NetFirewallRule`/
  `Get-ScheduledTask`/`Get-NetTCPConnection`, ставить
  `$ProgressPreference = 'SilentlyContinue'`.

### Что установлено (verified 03.09.2026)

- **CMSV6 `7.36.1_20251023`** (не та версия, что на сервере 1 — задача
  требовала именно эту сборку с Яндекс.Диска). Путь установки —
  `C:\Program Files\CMSServerV6\` (в этой сборке файлы лежат плоско, без
  версионной подпапки вида `\7.36.1_20251023\`, в отличие от текущей
  раскладки сервера 1 — см. предупреждение о переустановках выше).
- 🔴 **Как реально получилось рабочее состояние — не повторяй частичный
  путь.** Тихий инсталлятор (`/VERYSILENT /SUPPRESSMSGBOXES /NORESTART /SP-`)
  копирует только файлы, sha256/размер которых сверены на СКАЧАННОМ файле —
  Windows-службы и MySQL он не создаёт вообще. Первый проход GUI-мастера
  (`gpssvrwizard.exe`/`gpssvrctrl.exe` из `CMSServerV6\bin`), пройденный
  человеком через RDP, зарегистрировал большинство `GPS*`-служб, но НЕ
  `GPSMysqld` и не создал схему `1010GPS` — `tomcat` при этом не мог
  стартовать вообще, потому что `tomcat\conf\server.xml` в поставке
  отсутствует (см. ниже), а без БД веб-платформа зависала на инициализации
  (`TldScanner` в `catalina.log`, без прогресса). **Этот частичный путь
  рабочего сервера не даёт** — для сервера 3 его не воспроизводить. Рабочее
  состояние получилось только после того, как Максим **полностью
  переустановил CMSV6 вручную через RDP тем же инсталлятором** («я сам
  переустановил cmsv6 нужный и запустил он работает», 03.09.2026) — после
  этого появилась служба `GPSMysqld`, инициализировалась схема `1010GPS`, все
  службы вышли в `Running`. Версия и раскладка ПОСЛЕ этой переустановки
  подтверждены заново, а не только sha256 файла до неё:
  `C:\Program Files\CMSServerV6\bin\version.ini` → `version=7.36.1_20251023`;
  путь по-прежнему плоский, без версионной подпапки.
- Службы (все, кроме `GPSDaemon`, — `Manual`): `GPSDaemon` (Automatic),
  `GPSDataProcSvr`, `GPSDownSvr`, `GPSFtpd`, `GPSGatewaySvr`, `GPSGeocodeSvr`,
  `GPSLoginSvr`, `GPSMediaSvr`, `GPSMysqld`, `GPSStorageSvr`, `GPSUserSvr`,
  `gpstomcat6` — все `Running`.
- **Tomcat на порту 80.** Особенность этой сборки: `tomcat\conf\server.xml`
  в базовой поставке отсутствует вовсе — вместо него лежит набор именованных
  шаблонов (`server - 80.xml`, `server - 8080.xml`, `server - 443-https.xml`
  и другие), и активный `server.xml` нужно выбрать копированием одного из них
  — сам инсталлятор этого не делает. Скопирован `server - 80.xml` поверх
  `server.xml` (коннектор `Http11Nio2Protocol` на `port="80"`, без
  `address=127.0.0.1`); текущий `server.xml` на машине по-прежнему содержит
  этот коннектор (проверено заново после финальной переустановки). Локально и
  снаружи `http://200.165.238.242/` отдаёт `200`.
- **MySQL** — служба `GPSMysqld`, бинарь плоско в `mysql\bin\gpsmysqld`
  (подпапки `mysql\5.5_x64\`/`mysql\5.7_x64\`, оставшиеся от первого,
  незавершённого прохода мастера, не используются — актуальный `my.ini`
  находится в `mysql\my.ini`), слушает `127.0.0.1:3311`. `my.ini`:
  `basedir="C:/Program Files/CMSServerV6/mysql/"`,
  `datadir="C:/Program Files/CMSServerV6/mysql/gserver_dbdata/"`, `port=3311`,
  `bind-address=127.0.0.1` — та же раскладка, что на сервере 1. Схема
  `1010GPS`, `server_info` → `ID=3` (`IDNO='M1'`, `Name='Media Server'`),
  перепроверено после финальной переустановки:
  `IPClient`/`LanIP`/`IPClient2`/`IPDevice2` уже равны `200.165.238.242` —
  мастер установки проставил это сам при установке, правка не потребовалась.
  **verified 03.09.2026 (IPRSV1-18, вторая спека):** `IPClient`/`LanIP`
  строки `ID=3` переписаны на `scan-vision.ru` (как на сервере 1 — плеер под
  HTTPS должен получать доменное имя, покрытое сертификатом, а не голый IP);
  `IPDevice`/`PortDevice`/`PortClient`/`IPClient2`/`IPDevice2` не менялись.
  Дамп таблицы перед правкой снят на `C:\Users\Administrator\Desktop\`.
  `GPSLoginSvr` перезапущен после правки. У этой версии схемы `server_info`
  нет колонки `IsHttps` (была в спеке по аналогии с сервером 1 — в этой
  сборке её просто нет, не путать с отсутствием значения). Таблица
  `jt808_system_params` **есть** (52 строки, verified 03.09.2026), но строки
  `HttpsMapHttpPort` в ней нет (на сервере 1 это `id=61`) — на этой схеме
  такого параметра не завели вовсе, а не просто не заполнили. Значение
  подтверждено прямым SQL-запросом; проверка через
  `http://127.0.0.1:6605/3/1?...DevIDNO=...` по спеке — на сервере ещё нет ни
  одного зарегистрированного устройства, поэтому эндпоинт отвечает ошибкой
  параметров, а не отдаёт `server.clientIp`/`lanip`; сам факт в БД это не
  отменяет, довешивание этой проверки — когда появится первый реальный
  регистратор.
- **frps `0.17.0`** в `C:\frps\frps.ini` (формат INI, `[common]`):
  `bind_port=7000`, `vhost_http_port=9966`, `vhost_https_port=9967`,
  `tcp_mux=true`, `dashboard_addr=127.0.0.1`, `dashboard_port=7500`,
  `dashboard_user=admin`, `allow_ports=65535`, `max_ports_per_client=1`,
  `log_file=C:/frps/frps-run.log` (прямые слэши — обратные ломают парсер),
  `log_way=file`, `log_level=info`, `log_max_days=30`.
  🔴 **verified 03.09.2026 (IPRSV1-18, вторая спека) — изменение политики
  против первого прохода:** `subdomain_host` переписан с `scanvision.online`
  на **`scan-vision.ru`** (у сервера 2 теперь свой домен с рабочей
  wildcard-записью); `token` и `dashboard_pwd` больше **не собственные** —
  по прямой просьбе Максима приведены к значениям сервера 1 (прочитаны там
  по SSH, только чтение, сервер 1 не менялся). Следствие, которое нужно
  держать в голове: **общий токен на двух хостах означает, что компрометация
  любой из машин роняет обе, а ротация теперь всегда парная** — меняется на
  обеих машинах в один заход, иначе часть парка регистраторов отвалится.
  Значения — только в `frps.ini` на обеих машинах, нигде больше. Бэкап
  ini-файла до правки: `C:\frps\frps.ini.bak-before-iprsv1-18-parity`.
  Применено через задачу (`Stop-ScheduledTask`/`Start-ScheduledTask` frps),
  не запуском `frps.exe` руками.

### nginx (verified 03.09.2026, IPRSV1-18 вторая спека)

`C:\nginx` — nginx `1.30.4` (та же версия, что на сервере 1), скачан и
распакован заново (не скопирован с сервера 1). Слушает **443, 16604, 16605**
(порт 80 остаётся у CMSV6 — точку входа на nginx не переносили, это уже
делали 14.08.2026 и откатили по требованию Максима). `C:\nginx\conf\nginx.conf`
— копия конфигурации сервера 1 с заменой имён и путей сертификата: один
443-блок (`server_name scan-vision.ru www.scan-vision.ru`) с полным набором
websocket-директив IPRSV1-15 в общем `location /` (без них `Upgrade:
websocket` через 443 отдаёт `404` — подтверждено `101` в проверке ниже) и
два медиа-блока IPRSV1-17 (`16605 → 127.0.0.1:6605` `GPSLoginSvr`,
`16604 → 127.0.0.1:6604` `GPSMediaSvr`), без `ssl_stapling` (только для
сертификата GlobalSign сервера 1). На уровне `http {}` —
`client_max_body_size 512m;` и `map $http_upgrade $connection_upgrade`. Нет
`listen 80`, нет `location /.well-known/acme-challenge/` (challenge отдаёт
tomcat, см. win-acme ниже), нет HSTS, нет редиректа 80→443.

- `C:\nginx\run_nginx.bat` — идемпотентный (`tasklist` на живой `nginx.exe`,
  запускает только при отсутствии); задача планировщика `nginx_run`, триггер
  «при старте системы», от `SYSTEM`, `RunLevel Highest`, рабочая папка
  `C:\nginx`, перезапуск при сбое 1 минута × 3. Второй задачи под nginx нет.
- `C:\nginx\reload-nginx.bat` — две строки (`cd /d C:\nginx` +
  `nginx.exe -s reload`), вызывается win-acme после продления.
- Три ловушки сервера 1 воспроизведены один в один и подтверждены здесь:
  `nginx.exe -t` из SSH-сессии падает на относительных путях — запускать
  `nginx.exe -p C:\nginx -c conf\nginx.conf -t`; `nginx.exe -s reload` из
  интерактивной SSH-сессии (`Administrator`) не работает
  (`OpenEvent ... 5: Access is denied`), т.к. мастер запущен задачей от
  `SYSTEM` — применять через `taskkill /F /IM nginx.exe` +
  `schtasks /run /tn nginx_run`; после каждого перезапуска проверять
  `Get-Process nginx` (`StartTime`) и что на 443/16604/16605 слушают только
  свежие PID.

### win-acme и сертификат (verified 03.09.2026, IPRSV1-18 вторая спека)

`C:\wacs` — win-acme `2.2.9.1701` (pluggable, та же версия, что на сервере
1), скачан заново из официального релиза, не скопирован. Валидация —
`http-01` через файловую систему, webroot — корень веб-приложения tomcat
`C:\Program Files\CMSServerV6\tomcat\webapps\gpsweb` (найден через `Context
docBase="../webapps/gpsweb" path=""` в `server.xml`; `appBase="ttxapps"` из
`<Host>` пустой и не используется — не путать одно с другим при следующей
переустановке). Порт 80 у CMSV6 не отбирался, в `nginx.conf` ACME-location
не заводили.

- Сертификат выпущен на `scan-vision.ru` + `www.scan-vision.ru`, хранилище
  pemfiles в `C:\nginx\conf\le` — фактические имена файлов:
  `scan-vision.ru-chain.pem` / `scan-vision.ru-key.pem` (используются в
  `nginx.conf`), плюс `scan-vision.ru-crt.pem` и
  `scan-vision.ru-chain-only.pem`. Действителен `03.09.2026 → 02.12.2026`.
  Не wildcard: устройства-субдомены (`<devid>.scan-vision.ru`) сертификатом
  не покрыты, штатный путь к ним — обычный HTTP на `:9966` (как на
  сервере 1).
- Задача планировщика `win-acme renew (acme-v02.api.letsencrypt.org)`
  создана автоматически, включена, идёт от `SYSTEM`, старт `09:00` + случайная
  задержка до 4 часов. Следующее продление после **28.10.2026**.
- Принудительное продление (`wacs.exe --renew --force`) прогнано **один раз**
  03.09.2026 (недельный лимит Let's Encrypt на этом наборе имён — второй раз
  не гонять без крайней необходимости): использовало кеш заказа (в пределах
  1 суток после выпуска не расходует лимит повторно), экспортировало
  `.pem`-файлы заново. Попытка reload из самого win-acme (запущенного
  интерактивно по SSH от `Administrator`) закономерно упала с той же
  `OpenEvent ... Access is denied`, что и у сервера 1 — это ожидаемо и не
  баг: настоящее плановое продление идёт от задачи `win-acme renew (...)`,
  которая, как и `nginx_run`, работает от `SYSTEM` и поэтому применяет
  `reload-nginx.bat` без этой проблемы. После этого nginx перезапущен штатным
  способом (`taskkill` + `schtasks /run /tn nginx_run`), новые PID слушают
  443/16604/16605, `https://scan-vision.ru/` и `https://www.scan-vision.ru/`
  отдают `200` снаружи без `-k`.
- ⚠️ **Живое видео под HTTPS, вероятно, не работает** — как на сервере 1
  (флаг `IsHttps` там не решён, вопрос к вендору), а в схеме сервера 2
  колонки `IsHttps` нет вовсе (см. MySQL выше) и строки `HttpsMapHttpPort` в
  `jt808_system_params` тоже нет. Полный end-to-end путь плеера не проверен
  — на сервере ещё нет зарегистрированных устройств, чтобы дойти до реального
  video/archive запроса. Ожидаемое поведение: страница и интерфейс по HTTPS
  работают, видео под `https:` заблокируется как смешанное содержимое,
  рабочий путь для видео — `http://scan-vision.ru`. Известное ограничение,
  не повод откатывать задачу.

### Автозапуск (verified 03.09.2026)

Тот же механизм, что на сервере 1 (`C:\frps\run-frps.bat`, задачи `frps` +
`frps-watchdog`), воспроизведён с нуля:

- `C:\frps\run-frps.bat` — идемпотентный (`tasklist` на живой `frps.exe`,
  запускает только если его нет);
- задача `frps`: триггеры «при старте системы» + каждые 5 минут,
  `ExecutionTimeLimit PT0S` (без лимита), `MultipleInstances IgnoreNew`,
  `RestartOnFailure` 1 минута × 3, от `SYSTEM`;
- задача `frps-watchdog`: каждые 5 минут (сдвиг +2 мин от задачи `frps`),
  `ExecutionTimeLimit PT5M`, от `SYSTEM`; ничего не делает, пока `frps.exe`
  жив, иначе останавливает задачу `frps` (если она ещё `Running`) и поднимает
  заново через `schtasks /run /tn frps`.
- Проверено принудительным `taskkill /F /IM frps.exe`: новый процесс поднялся
  через ~1 минуту (15:22:54 → 15:24:03 в тесте 03.09.2026), ровно один
  экземпляр, все порты снова слушаются.

### Firewall (verified 03.09.2026, IPRSV1-18 вторая спека)

97 → 99 включённых входящих allow-правил после добавления пары
`iprsv1-18-parity-*`, замерено `Get-NetFirewallRule -Direction Inbound
-Action Allow -Enabled True` (было 72 → 73 после первого прохода того же дня,
03.09.2026, метод того замера в спеке/CLAUDE.md не зафиксирован. Расхождение
между «73» и «97» не пересчитывалось назад и не объяснено — оба числа
реальные последовательные замеры одного дня, а не ошибка счёта). Правила:

- `iprsv1-18-cmsv6-tcp` — TCP **80, 2121-2162, 6601-6612, 6617, 6630-6635**
- `iprsv1-18-cmsv6-udp` — UDP **6601-6612**
- `iprsv1-18-frps-tcp` — TCP **7000, 9966, 9967, 20021**
- `iprsv1-18-443-tcp` — TCP **443**
- `iprsv1-18-parity-tcp` (новое, 03.09.2026) — TCP **88, 8080, 8088, 16601,
  16603-16605, 16607-16609, 16611, 20000-21000, 30000-31000**
- `iprsv1-18-parity-udp` (новое, 03.09.2026) — UDP **20000-21000,
  30000-31000**
- `OpenSSH Server (sshd)` — TCP 22 (создано человеком в рамках шага 0)

Все — `-Profile Any -RemoteAddress Any -Program Any -Action Allow`. Порт 7500
(дашборд frps) не открыт и не должен быть — подтверждено снаружи (таймаут).
Динамический диапазон портов сервера 2 (`netsh int ipv4 show dynamicport`,
verified 03.09.2026): TCP и UDP **9000-64999** — совпадает с сервером 1, и
часть открытых здесь диапазонов (20000-21000, 30000-31000) попадает внутрь
него. Риск известный и на сервере 1 принят Максимом сознательно, замер
зафиксирован, а не предположен.

История: `iprsv1-18-443-tcp` и порт `20021` в правиле `iprsv1-18-frps-tcp`
были открыты ещё 03.09.2026, до этой задачи и до того, как за 443 появился
слушатель — по прямому запросу Максима в чате, вопреки первой спеке, которая
443 прямо запрещала (nginx/TLS были тогда вне задачи). Это отклонение с тех
пор реализовано: 443 теперь занят nginx.

### Отличия от сервера 1

- **Версия и раскладка CMSV6.** `7.36.1_20251023` против текущей версии
  сервера 1, плоская раскладка каталогов (`C:\Program Files\CMSServerV6\`
  без версионной подпапки) против вложенной на сервере 1.
- Служба `gpstomcat6` в этой сборке `Manual`, а не `Automatic` (не менялось
  специально, так поставил мастер).
- **Схема БД `1010GPS` отличается от сервера 1:** нет колонки `IsHttps` в
  `server_info`, нет строки `HttpsMapHttpPort` в `jt808_system_params` (сама
  таблица есть, строки нет). Из-за этого статус видео под HTTPS здесь нельзя
  привести к состоянию сервера 1 копированием — там тоже не работает
  (флаг `IsHttps` не решён), но по другой причине.
- **Сертификат.** Let's Encrypt (win-acme, автопродление) против купленного
  GlobalSign без автопродления на сервере 1 — сознательное решение этой
  задачи, а не временное состояние: у сервера 2 нет купленного сертификата,
  и заводить его не просили.

### Как перевести регистратор на этот сервер

В `frpc.ini`/`frpcSet.xml` на самом регистраторе поменять **только**
`<ServerIp>` на `200.165.238.242` — `<TokenForCon>` менять не нужно: с
03.09.2026 `token` frps общий для обоих серверов (см. frps.ini выше).
Один `<ServerIp>` без смены токена теперь подключится сам. Это и есть цена
общего токена: удобство переключения регистратора в обмен на парную
ротацию при компрометации любой из машин. `subdomain_host` у серверов
по-прежнему разный (`scanvision.online` на сервере 1 против
`scan-vision.ru` на сервере 2) — публичное имя регистратора после переезда
меняется на `<devid>.scan-vision.ru`, сертификатом оно не покрыто, путь к
нему обычный HTTP на `:9966`.
