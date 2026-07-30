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

## Secrets

Never write actual values of `LOGIN`, `PASSWORD`, `SERVER`, `token`, or
`dashboard_pwd` into this repo, PR descriptions, commit messages, or command
logs — only their names and how they're used.
