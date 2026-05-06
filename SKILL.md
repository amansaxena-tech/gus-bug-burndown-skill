---
name: gus-bug-burndown
description: Fetch P0/P1/P2 bug burndown data for a Salesforce release across THB programs. Queries GUS for bugs per project and renders a weekly line chart (opened, closed, net open) as an interactive HTML report — one chart per project plus an all-projects combined view.
---

# GUS Bug Burndown

Use this skill when the user wants a P0/P1/P2 bug burndown chart for a release. It fetches data from GUS, aggregates bugs by project and week, and renders an interactive HTML line chart.

## Programs in scope

Only include work items mapped to these THB programs (product tags):

- [262] THB - Voice Infrastructure
- [262] THB - Transcriptions
- [262] THB - Queuing & Routing, IVA, IVR, Media File Mgmt
- [262] THB - Metering and Billing, License Enablement
- [262] THB - HA, Compliance and BCP
- [262] THB - Digital Voice
- [262] THB - Analytics & Recording
- [262] THB - Agent Experience
- [262] THB - Admin Setup

When the user provides a different release number (e.g. 264), replace `262` with that number in all product tag names above.

## Reference files

- **query.md** — SOQL patterns, product tag list, weekly bucketing, and `compute_series` logic.
- **chart.md** — How to generate the HTML line chart output.

## Workflow

1. Ask the user for the release name/number if not already provided.
2. Run preflight: verify GUS auth.
3. Resolve all product tag IDs for the release's THB programs (see query.md).
4. Fetch all P0/P1/P2 bugs by product tag — **no date filter, no build filter** (see query.md).
5. Bucket bugs by project and week using `CreatedDate` (opened) and `LastModifiedDate` (closed) — see query.md.
6. Compute three series per project across the last 6 weeks using `compute_series` from query.md.
7. Generate the HTML line chart (see chart.md) — combined all-projects panel first, then per-project panels grouped by program.

## Core rules

- All `sf` commands must include `--target-org gus`.
- Never hardcode IDs — resolve them at runtime via SOQL.
- **Record type filter**: `Type__c = 'Bug'` and `Priority__c IN ('P0', 'P1', 'P2')`.
- **No build or date filter on the bug query** — scope is all-time P0/P1/P2 bugs on the product tag. Adding a build/date filter produces incorrect, incomplete counts (verified against GUS).
- **Do not query via epics** — THB epics contain User Stories not bugs; the epic path returns near-zero bug results.
- **Three lines per chart**: Opened this week (blue), Closed this week (green), Net Open cumulative (red).
- Net Open must be computed from full history, not just the display window, so pre-display backlog is included.
- Always show **last 6 weeks** on the X-axis.
- Cross-verify totals by querying GUS directly. Do not rely solely on Slack/slackbot — it may have caching lag of 1-3 bugs.
- If the release report URL is provided, use it only to confirm program/project names and scope — not for bug counts.

## Authentication

If `sf data query --target-org gus` fails:
```bash
sf org login web --instance-url https://gus.my.salesforce.com --alias gus
```
