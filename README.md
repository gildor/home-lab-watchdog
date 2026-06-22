# home-lab-watchdog

External Tailscale reachability probe for a private home lab. A scheduled
GitHub Action joins the tailnet (ephemeral node) and checks home is reachable
over Tailscale; on success it pings a [healthchecks.io](https://healthchecks.io)
check (**External to Home**), which alerts if the pings stop.

This is the **only** monitor that tests the *inbound* path — "can an outside
machine reach home over Tailscale". The in-house monitors (Uptime Kuma,
net-watchdog) all probe from *inside* the NAS and can't see this.

It probes **two paths to the same Caddy**, because they fail independently:

- **`HOME_TARGET`** — a home **tailnet `/32`** (e.g. `home-nas`). Tests tailnet
  auth + DERP/relay + the node + Caddy.
- **`HOME_LAN_TARGET`** — a home **LAN IP** (e.g. `192.168.50.175`) reached
  **through the subnet route** (the node joins with `--accept-routes`). Tests the
  primary-subnet-router election + forwarding.

healthchecks is pinged **only when both are reachable**, so **External to Home**
goes red on node death **or** a subnet-route hijack. The second path exists
because on **2026-06-22** the HA Tailscale add-on (userspace mode) won the
`192.168.50.0/24` primary-router election and blackholed the whole LAN, yet the
old `/32`-only probe stayed green throughout — it routed around the very thing
that broke.

## The probe is hardened against transient blips

A single probe occasionally clips on Tailscale DERP-relay jitter or a brief WAN
hiccup, so the workflow retries up to **4× in-run** and pings healthchecks the
instant **both paths** are up together. It only stays silent (→ healthchecks
alerts) when it never sees both green — preserving the "green can't lie"
dead-man property.

## Gotcha: GitHub's scheduler is unreliable — tune the HC grace, not the cron

GitHub Actions `schedule:` crons are **best-effort**. Under load GitHub silently
delays and coalesces scheduled runs: with a `*/30` cron this repo actually fires
**~every 2–2.5h**, with occasional 2.5h+ gaps. Bumping the cron frequency does
**not** fix this — GitHub ignores it (observed 2026-06).

So the **External to Home** healthchecks check is deliberately given a **wide
grace** to absorb GitHub's lateness, rather than trying to make GitHub punctual:

- Current tuning: `timeout 2h + grace 2h` = **4h** of silence before it alerts.
- Real outages are still caught in ~10min by the *inside* net-watchdog checks
  (Internet Home / lan-access). **External is a coarse backstop** for the inbound
  Tailscale path specifically, where slow detection is acceptable.
- The grace lives in **healthchecks.io** (check `External to Home`), set via its
  API or UI — **not** in this repo.

A flap of **External to Home** almost always means *GitHub didn't run the job in
time*, not that home was unreachable. **Cross-check the inside checks before
worrying** — if Internet Home / lan-access stayed green, home was fine.

### If it still false-alarms (a GitHub >4h stall): make runs reliable

Don't widen the grace into uselessness. Instead, stop relying on GitHub's scheduler
and trigger the run via **`workflow_dispatch`** from a reliable external cron (e.g.
cron-job.org calling the GitHub API every ~20–30min). Dispatched runs are **not**
coalesced — they fire within seconds. Then tighten the HC grace back to ~70min for
fast detection.

Keep the trigger **external** (not the NAS) so the check stays independent of
home-side infra — a NAS-driven trigger would make "can the outside reach home"
secretly depend on the NAS being up. Token: a **fine-grained PAT** scoped to this
repo, `Actions: write` only (public repo → negligible blast radius; max 1y, rotate
yearly).

## keepalive

GitHub auto-disables scheduled workflows after 60 days of no repo activity; the
`keepalive` workflow commits a timestamp twice a month to keep the schedule enabled.
