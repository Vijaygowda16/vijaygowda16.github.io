+++
title = "Building a Multi-Region Disk Usage Dashboard"
date = "2025-06-01"
author = "Vijay Kumar C S"
tags = ["infrastructure", "python", "monitoring", "aws", "devops"]
description = "How I built an internal disk usage monitoring dashboard across multiple AWS regions with per-user analytics, volume breakdowns, and automated email alerts — all served as a self-contained HTML file."
showFullContent = false
readingTime = true
hideComments = false
+++

## Overview

Managing storage across a distributed infrastructure is painful without visibility. When engineering teams share NFS/EFS volumes across multiple AWS regions, runaway home directories and scratch volumes are nearly impossible to catch early — without a centralized view.

This post documents how I built a **Disk Usage Dashboard**: an interactive, self-contained monitoring tool generated entirely in Python and served via Apache, with per-user drill-downs, per-volume breakdowns, and automated email alerting.

---

## The Problem

A typical multi-region storage setup might look like this:

| Region | Code | Volume Types |
|--------|------|--------------|
| US West | `USW2` | home, scratch, project, archive |
| Asia Pacific | `APS1` | home, scratch |
| EU Central | `EUC1` | home, scratch |

Engineers fill up scratch volumes without realizing it. Home directories balloon over time. The only recourse — without tooling — is SSHing into each region and running `du` manually. That doesn't scale.

**Requirements I set for myself:**

- Single-page dashboard, no external dependencies at runtime
- Per-volume and per-user breakdown with drill-down modals
- Multi-region support with a clean region switcher
- Automated cron-based data collection
- Email alerts when volumes approach capacity thresholds

---

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                      Data Collection                         │
│                                                             │
│  USW2 cron job ──┐                                          │
│  APS1 cron job ──┼──► JSON snapshots ──► generate_dashboard.py │
│  EUC1 cron job ──┘                           │              │
│                                              ▼              │
│                                      dashboard.html         │
│                                      (self-contained)       │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
                    Apache alias → /dashboard/
                    internal.example.com/diskusage/dashboard/
```

The pipeline has three stages:

1. **Collection** — cron jobs on each region's login node run `du` against each NFS/EFS mount and emit structured JSON
2. **Generation** — `generate_dashboard.py` reads all region snapshots and renders a single, self-contained `dashboard.html` with all data embedded inline
3. **Serving** — Apache serves the file via an alias; no additional web server config needed

---

## Data Collection

Each region runs a lightweight collection script on a schedule. The script walks each volume mount, aggregates per-user usage via `du -s`, and writes a timestamped JSON snapshot:

```json
{
  "region": "USW2",
  "collected_at": "2025-05-28T02:00:00Z",
  "volumes": {
    "home": {
      "total_gb": 4096,
      "used_gb": 2341,
      "users": {
        "alice": 88.2,
        "bob": 42.5,
        "carol": 4.1
      }
    },
    "scratch": {
      "total_gb": 8192,
      "used_gb": 6100,
      "users": {
        "alice": 512.0,
        "dave": 3200.8
      }
    }
  }
}
```

One early challenge was **region-specific mount path conventions** — some regions used different NFS mount prefixes, causing the collection script to silently skip volumes. Fixed by normalizing mount point resolution before the `du` walk and adding explicit verification.

---

## Dashboard Generator

`generate_dashboard.py` is the core of the system. It reads all JSON snapshots and produces a single HTML file with:

- All CSS inlined in a `<style>` block
- All JavaScript inlined in a `<script>` block
- All data embedded as a JSON literal: `const DATA = {...}`

This makes the dashboard fully portable — copy the HTML file anywhere and it works with no build step, no server, no dependencies.

### Key Features

**Region Switcher**  
A tab-style selector at the top filters all charts and tables to the selected region. Implemented with vanilla JS toggling CSS display classes — no framework needed.

**Per-Volume Modal**  
Clicking any volume card opens a modal with a bar chart showing each user's contribution to that volume. Built with Chart.js rendered inline.

**Per-User Modal**  
Clicking a username in any table opens a modal showing that user's footprint *across all volumes* in the current region — useful for finding engineers with forgotten large scratch allocations.

**Threshold Indicators**  
Volumes above 80% capacity get an amber badge; above 90% get a red badge. These same thresholds drive the email alerts.

---

## Email Alerting

`send_alerts.py` runs after each data collection cycle and checks every volume against configured thresholds. When a volume crosses a threshold, it generates an alert email listing:

- Volume name and region
- Current usage percentage
- Top 5 users by consumption
- Trend direction (growing / stable / shrinking) based on the last 7 snapshots

```python
THRESHOLDS = {
    "warn":     0.80,   # 80% — amber alert
    "critical": 0.90,   # 90% — red alert
}

ALERT_RECIPIENTS = ["infra-team@example.com"]
```

The alerting logic is deliberately stateful — it tracks which volumes have already triggered an alert to avoid sending repeat emails on every collection cycle until the volume drops back below threshold.

---

## Cron Schedule

Collection and generation are coordinated across regions with staggered start times to avoid all regions hitting shared storage simultaneously:

```cron
# USW2 — data collection
0 2 * * * /opt/diskusage/collect.sh usw2

# APS1 — offset by 20 minutes
20 2 * * * /opt/diskusage/collect.sh aps1

# EUC1 — offset by 40 minutes
40 2 * * * /opt/diskusage/collect.sh euc1

# Dashboard generation — runs after all regions complete
0 3 * * * /opt/diskusage/generate_dashboard.py
```

---

## Deployment

The dashboard is served from a shared directory via an existing Apache alias — no new infrastructure required.

```apache
Alias /diskusage/ /data/shared/diskusage/
<Directory /data/shared/diskusage/>
    Options Indexes FollowSymLinks
    AllowOverride None
    Require all granted
</Directory>
```

`generate_dashboard.py` writes directly to `dashboard.html` in that directory. Apache picks up the new file immediately — no restarts, no cache invalidation needed.

---

## Challenges & Lessons

**Silent collection failures** — In some regions, the collection script was silently failing when NFS mount paths didn't match expected conventions. Added explicit mount verification before the `du` walk; the cron job now emails on collection failure rather than silently producing stale data.

**`du` is slow on large volumes** — Running `du -s /home/*` on a large home volume with thousands of subdirectories was taking 8–12 minutes. Switched to `du --max-depth=1` with a configurable timeout and a fallback to the previous snapshot if collection exceeds the time limit.

**Self-contained HTML size** — With Chart.js inlined, the generated HTML was ~400KB. Acceptable for an internal tool, but added a simple minification pass on the Chart.js bundle in the generator to bring it down to ~180KB.

**Stale data visibility** — Users initially had no way to know how old the dashboard data was. Added a "Last updated" timestamp in the header, pulled from the most recent snapshot's `collected_at` field.

---

## What's Next

- **7-day trend sparklines** — showing volume growth/shrink direction inline in the cards
- **Quota enforcement integration** — surfacing configured quotas alongside actual usage
- **Slack notifications** — complement email alerts with a channel message for critical-threshold breaches
- **Historical retention** — keeping 30 days of snapshots to support trend analysis

---

## Project Structure

```
diskusage/
├── collect.sh              # Per-region collection wrapper
├── collect_volume.py       # Core du aggregation logic
├── generate_dashboard.py   # HTML generation pipeline
├── send_alerts.py          # Threshold alerting
└── snapshots/
    ├── usw2_latest.json
    ├── aps1_latest.json
    └── euc1_latest.json
```

---

*A lightweight alternative to heavyweight monitoring stacks for teams that just need storage visibility without the operational overhead.*
