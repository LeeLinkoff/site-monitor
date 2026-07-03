# Site Monitor (GitHub Actions)

Lightweight uptime monitoring using GitHub Actions with email and ntfy alerts.

---

## Purpose

Provide an independent, external monitoring signal outside the hosting provider.

This system is designed as:

- a lightweight monitoring solution
- a backup to external/free uptime monitoring services
- a demonstration of stateful alerting using GitHub Actions

---

## Why GitHub Actions

GitHub Actions is a well known, documented approach for free external uptime monitoring. It runs on infrastructure completely independent of your hosting provider, which matters for a few reasons:

- it is free with no commercial dependency
- commercial uptime monitoring services can go down, change pricing, or cease to exist (GitHub Actions has none of those risks)
- it runs outside your hosting environment, so if your host goes down, the monitor still runs
- the pattern is widely used and recognized in the developer community

This is a deliberate architectural choice, not a workaround.

---

## How It Works

- A scheduled GitHub Actions workflow runs hourly
- TARGET_URL can hold one or more URLs, comma-separated (example: `https://leelinkoff.com,https://nuvrix.io`)
- The workflow sends an HTTP request to each URL independently
- If a response is not HTTP 200, that URL is marked DOWN, the rest are unaffected
- Previous state per URL is persisted in a dedicated `state` branch
- Alerts are sent based on state transitions and current status, evaluated separately for each URL

---

## Quick Status Check

Two ways to check status. Both are read-only, neither requires running the workflow or opening a terminal.

### Method 1: raw status.txt (fastest, shows per-URL detail)

1. Open this URL directly in any browser, including mobile:

```
https://raw.githubusercontent.com/LeeLinkoff/site-monitor/state/status.txt
```

2. The page shows one line per monitored URL, formatted as `URL=STATUS`. Example with two targets:

```
https://leelinkoff.com=UP
https://nuvrix.io=DOWN
```

3. Read it top to bottom, each line is independent, one URL being DOWN does not mean the others are.

No login required. Bookmark this URL for one-tap checking.

Limitations:

- This is only as current as the last successful scheduled run. If the workflow hasn't executed (see the disabled-workflow note under Limitations below), this file stops updating and everything on it goes stale, it will keep showing the last real result forever until the workflow runs again.
- If the `state` branch or `status.txt` does not exist yet (before the very first run), this URL returns a 404 page instead of status text.

### Method 2: Actions tab (confirms the workflow itself ran, not just site status)

1. Go to `https://github.com/LeeLinkoff/site-monitor/actions`
2. In the left sidebar, click "Site Monitor Check"
3. Look at the most recent run in the list: a green check mark means the workflow completed without error, a red X means it failed
4. Click into that run, then click the "check" job, to see the full `[INFO]` log lines per URL from the run itself

This method answers a different question than Method 1. A green check here means the workflow ran and finished, it does not by itself tell you whether any individual URL was UP or DOWN, you still need to open the job log or check status.txt for that. A red X means something in the workflow broke (bad credentials, network failure, git push failure), separate from whether your sites are actually reachable.

---

## Alerting Logic

These rules are evaluated separately for each URL in TARGET_URL:

```
UP → UP     → no alert
UP → DOWN   → alert
DOWN → DOWN → alert every scheduled run
DOWN → UP   → alert (recovery)
```

If any single URL needs an alert this run, one consolidated email/ntfy message is sent listing the current status of every monitored URL, one line each:

```
<url>: UP
<url>: DOWN
```

Example, if only `nuvrix.io` went down while `leelinkoff.com` stayed up, the alert still lists both:

```
<https://leelinkoff.com>: UP
<https://nuvrix.io>: DOWN
```

---

## Schedule

```
cron: "17 * * * *"
```

- Runs once per hour at minute 17 (UTC)
- GitHub Actions scheduling is best-effort
- Execution timing is not guaranteed to be exact

---

## Components

- GitHub Actions (scheduler and execution)
- curl (HTTP checks)
- msmtp (email alerts via Gmail)
- ntfy (push notifications)
- `state` branch (state persistence)

---

## Configuration

Note: navigation and page layout below confirmed accurate as of this README update (July 3, 2026). GitHub's Settings UI has changed layout before and may change again; if the path below doesn't match what you see, that's a UI change, not an error on your part.

All values below are set in one place:

```
https://github.com/LeeLinkoff/site-monitor/settings/secrets/actions
```

Or navigate manually: repo > Settings > Secrets and variables > Actions. The page itself is titled "Actions secrets and variables" and has two separate tabs on it, Secrets and Variables. They are not interchangeable and the workflow reads them differently.

### Variables tab

- TARGET_URL, value shown in plain text on this page, also printed unmasked in the "Debug configuration" step log of any run
- Accepts one URL, or multiple URLs separated by commas, no spaces required around commas (`https://a.com,https://b.com` and `https://a.com, https://b.com` both work, the workflow trims whitespace)

### Secrets tab

- EMAIL_TO
- GMAIL_USER
- GMAIL_PASS (Gmail App Password, not account password)
- NTFY_TOPIC
- Values are write-only once saved. GitHub will not display them again on this page or in any log (masked as `***` even if echoed).

### How these map into monitor.yml

Both tabs feed the same `env:` block in `.github/workflows/monitor.yml`, using two different lookup syntaxes:

```
env:
  TARGET_URL: ${{ vars.TARGET_URL }}     # from the Variables tab
  EMAIL_TO: ${{ secrets.EMAIL_TO }}      # from the Secrets tab
  GMAIL_USER: ${{ secrets.GMAIL_USER }}  # from the Secrets tab
  GMAIL_PASS: ${{ secrets.GMAIL_PASS }}  # from the Secrets tab
  NTFY_TOPIC: ${{ secrets.NTFY_TOPIC }}  # from the Secrets tab
```

`vars.X` pulls from Variables, `secrets.X` pulls from Secrets. If a name is added under the wrong tab (for example, TARGET_URL saved as a Secret instead of a Variable), the workflow's "Debug configuration" step fails with `[ERROR] TARGET_URL is not set`, since `vars.TARGET_URL` would come back empty even though a value exists under the Secrets tab.

---

## Monitoring Scope

This setup verifies:

- Site availability (HTTP response)
- Endpoint reachability

This setup does not verify:

- Application correctness
- Performance or latency
- Partial failures returning HTTP 200

---

## State Persistence

Previous status per URL is stored in a dedicated `state` branch in this repository. This branch contains only `state/status.txt` and is completely separate from `main`. The `main` branch history stays clean (no automated state commits ever appear on `main`).

status.txt holds one line per monitored URL, formatted as `URL=STATUS`. A URL not yet present in this file (first time it is added to TARGET_URL) defaults to UP until the first check runs.

This approach is more reliable than GitHub Actions cache, which is best-effort and can silently fail to persist between runs.

---

## Limitations

- GitHub Actions is not a real-time scheduler
- Execution timing may drift
- On the very first run, no previous state exists and defaults to UP

### ntfy iOS App (Message History)

- The ntfy iOS app does not reliably display message history
- This is a known limitation of the ntfy iOS app, not this workflow
- Notifications are delivered and cached correctly on the server
- To view full message history, open the following in a browser:

```
https://ntfy.sh/<your-topic>/json?poll=1&since=24h
```

---

## Usage

- Create a `state` branch in the repository before the first run
- Configure TARGET_URL
- Set required secrets
- Run the workflow manually once to validate alerts
- Enable scheduled runs

---

## Summary

This provides a cost-free, external uptime monitoring layer with:

- state-aware alerting
- repeated outage notifications
- recovery detection
- no commercial dependency

Best used as a backup monitor, not a primary production monitoring system requiring strict timing guarantees.
