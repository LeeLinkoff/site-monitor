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
- commercial uptime monitoring services can go down, change pricing, or cease to exist — GitHub Actions has none of those risks
- it runs outside your hosting environment, so if your host goes down, the monitor still runs
- the pattern is widely used and recognized in the developer community

This is a deliberate architectural choice, not a workaround.

---

## How It Works

- A scheduled GitHub Actions workflow runs hourly
- The workflow sends an HTTP request to the target URL
- If the response is not HTTP 200, the system marks the site as DOWN
- Previous state is persisted in a dedicated `state` branch
- Alerts are sent based on state transitions and current status

---

## Alerting Logic

```
UP → UP     → no alert
UP → DOWN   → alert
DOWN → DOWN → alert every scheduled run
DOWN → UP   → alert (recovery)
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

Set the following in repository settings:

### Variables

- TARGET_URL

### Secrets

- EMAIL_TO
- GMAIL_USER
- GMAIL_PASS (Gmail App Password, not account password)
- NTFY_TOPIC

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

Previous status is stored in a dedicated `state` branch in this repository. This branch contains only `state/status.txt` and is completely separate from `main`. The `main` branch history stays clean — no automated state commits ever appear on `main`.

This approach is more reliable than GitHub Actions cache, which is best-effort and can silently fail to persist between runs.

---

## Limitations

- GitHub Actions is not a real-time scheduler
- Execution timing may drift
- On the very first run, no previous state exists and defaults to UP

### ntfy iOS App — Message History

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
