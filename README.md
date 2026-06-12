# home-lab-watchdog

External **Tailscale reachability** probe for a private home lab. An hourly
GitHub Action joins the tailnet (ephemeral node) and checks a home service is
reachable over Tailscale; on success it pings a [healthchecks.io](https://healthchecks.io)
check, which alerts if the pings stop.

No secrets live in this repo — the Tailscale auth key, target host and ping URL
are GitHub Actions **secrets**. Public only so Actions minutes are free.
