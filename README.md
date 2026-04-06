# site-monitor
Hourly uptime monitor using GitHub Actions with email and ntfy push alerts. Stateful alerting with repeat down notifications and single recovery alert.

---

## Purpose

Provide an independent, external monitoring signal outside the hosting provider.

This system is designed as:

- a lightweight monitoring solution
- a backup to external/free uptime monitoring services
- a demonstration of stateful alerting using GitHub Actions

---

## How It Works

- A scheduled GitHub Actions workflow runs hourly
- The workflow sends an HTTP request to the target URL
- If the response is not HTTP 200, the system marks the site as DOWN
- Previous state is restored using GitHub Actions cache
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
- GitHub Actions cache (state persistence)

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

- Previous status is stored using GitHub Actions cache
- Cache uses dynamic keys with restore-keys to retrieve the latest state
- No repository commits or external storage required

---

## Limitations

- GitHub Actions is not a real-time scheduler
- Execution timing may drift
- Cache persistence is best-effort
- In rare cases (cache miss), previous state defaults to UNKNOWN
  - This may result in one extra alert

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

Best used as a backup monitor, not a primary production monitoring system requiring strict timing guarantees.
