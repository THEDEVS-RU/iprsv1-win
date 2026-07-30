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
  PowerShell and feed it a script on stdin — inline multi-line PowerShell
  passed as a single quoted argument breaks on quoting:
  ```bash
  ssh -p 2299 "$LOGIN@$SERVER" 'powershell -NoProfile -NonInteractive -Command -' <<'PS'
  ... multi-line PowerShell ...
  PS
  ```
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
- Dashboard binds to **127.0.0.1:7500 only** — never opened in the firewall.
  Reach it through an SSH tunnel:
  ```bash
  ssh -p 2299 -L 7500:127.0.0.1:7500 "$LOGIN@$SERVER"
  ```
  then open `http://127.0.0.1:7500`, user `admin`.
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
- `subdomain_host` is intentionally not set — the project has no domain yet.
  Until it's set, frps rejects any `subdomain`-based http/https proxy with
  "subdomain is not supported because this feature is not enabled by frps";
  that is expected, not a bug. To enable it later: point a wildcard A record
  at the address in `SERVER`, add `subdomain_host = <domain>` to `frps.ini`,
  then restart the `frps` scheduled task.

## Secrets

Never write actual values of `LOGIN`, `PASSWORD`, `SERVER`, `token`, or
`dashboard_pwd` into this repo, PR descriptions, commit messages, or command
logs — only their names and how they're used.
