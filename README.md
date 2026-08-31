# iprsv1-win

Operations documentation for the **IPRSV1** project server — a Windows Server
2022 VDS running the CMSV6 GPS/video platform behind nginx, plus an `frps`
reverse proxy used to reach in-vehicle recorders.

There is no application code here. The repository holds:

- **[CLAUDE.md](CLAUDE.md)** — the live state of the machine: how to connect,
  what runs where, which ports and certificates are in play, every gotcha
  found the hard way, and the currently open issues. Start here.
- **[specs/](specs)** — the task specs that produced that state, in
  chronological order. They are historical: read them for *why* something was
  done, not for what the server looks like today.

Last full verification sweep against the live server: **31.08.2026**.
