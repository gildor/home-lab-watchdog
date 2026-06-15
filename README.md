# home-lab-watchdog

External Tailscale reachability probe for a private home lab. A scheduled
GitHub Action joins the tailnet (ephemeral node) and checks a home service is
reachable over Tailscale; on success it pings a [healthchecks.io](https://healthchecks.io)
check (**External to Home**), which alerts if the pings stop.

This is the **only** monitor that tests the *inbound* path — "can an outside
machine reach home over Tailscale" (tailnet auth + relay/connectivity + the
`home-nas` subnet route + Caddy). The in-house monitors (Uptime Kuma, net-watchdog)
all probe from *inside* the NAS and can't see this.

## The probe is hardened against transient blips

A single probe occasionally clips on Tailscale DERP-relay jitter or a brief WAN
hiccup, so the workflow retries up to **4× in-run** and pings healthchecks the
instant any attempt succeeds. It only stays silent (→ healthchecks alerts) when
*all* attempts fail — preserving the "green can't lie" dead-man property.

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
