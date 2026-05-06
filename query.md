# GUS Bug Burndown — Queries

## Key rules learned from verification

- **Do NOT filter by build name or date range.** Bugs are scoped by product tag only. Adding a build/date filter produces incorrect, incomplete counts.
- **Do NOT query via epics.** Epics under PPM projects contain User Stories, not bugs. The Epic→Work path returns near-zero bugs.
- **Product tag is the correct scope.** Query `ADM_Work__c` directly with `Product_Tag__c IN (...)`.
- **Cross-verify totals against GUS directly**, not via Slack/slackbot — Slack may have caching lag of 1-3 bugs.

## Step 1: Resolve product tag IDs for the release's THB programs

Product tags for THB do NOT use bracket notation like `[262] THB - ...`. They use plain names like `ThunderBird - Admin Experience & Number Management`. Use the list below for Release 262. For other releases, search with `Name LIKE '%ThunderBird%' OR Name LIKE '%THB%'` and map manually.

**Release 262 product tags and their program mapping:**

| Product Tag Name | Program |
|---|---|
| ThunderBird - Admin Experience & Number Management | [262] THB - Admin Setup |
| ThunderBird - Agent Experience & Media Mgmt | [262] THB - Agent Experience |
| ThunderBird - Advance Quality Management | [262] THB - Analytics & Recording |
| ThunderBird - Native Call Recording | [262] THB - Analytics & Recording |
| ThunderBird - Recording and Transcription | [262] THB - Analytics & Recording |
| ThunderBird - Reporting and Analytics | [262] THB - Analytics & Recording |
| Thunderbird Digital Voice | [262] THB - Digital Voice |
| Thunderbird - Outbound | [262] THB - Digital Voice |
| THB Billing and Metering | [262] THB - Metering and Billing, License Enablement |
| ThunderBird - IVA - Agentforce Voice Integration | [262] THB - Queuing & Routing, IVA, IVR, Media File Mgmt |
| ThunderBird - IVA - CCAI | [262] THB - Queuing & Routing, IVA, IVR, Media File Mgmt |
| Thunderbird - IVR | [262] THB - Queuing & Routing, IVA, IVR, Media File Mgmt |
| ThunderBird  - Native IVR | [262] THB - Queuing & Routing, IVA, IVR, Media File Mgmt |
| ThunderBird - Media File Management | [262] THB - Queuing & Routing, IVA, IVR, Media File Mgmt |
| Thunderbird - MFM Core | [262] THB - Queuing & Routing, IVA, IVR, Media File Mgmt |
| ThunderBird - Omni Flow (Workflow Engine) | [262] THB - Queuing & Routing, IVA, IVR, Media File Mgmt |
| ThunderBird - Transcription | [262] THB - Transcriptions |
| ThunderBird - Native Voice - Team-1 | [262] THB - Voice Infrastructure |
| ThunderBird - Native Voice - Team-2 | [262] THB - Voice Infrastructure |
| ThunderBird - Native Voice - Team-3 | [262] THB - Voice Infrastructure |
| ThunderBird - Voice Infra - Signal Wire and Abstraction Layer (SCPV) | [262] THB - Voice Infrastructure |

Resolve IDs with:
```bash
sf data query --target-org gus --query "
  SELECT Id, Name
  FROM ADM_Product_Tag__c
  WHERE Name IN (
    'ThunderBird - Admin Experience & Number Management',
    'ThunderBird - Agent Experience & Media Mgmt',
    'ThunderBird - Advance Quality Management',
    'ThunderBird - Native Call Recording',
    'ThunderBird - Recording and Transcription',
    'ThunderBird - Reporting and Analytics',
    'Thunderbird Digital Voice',
    'Thunderbird - Outbound',
    'THB Billing and Metering',
    'ThunderBird - IVA - Agentforce Voice Integration',
    'ThunderBird - IVA - CCAI',
    'Thunderbird - IVR',
    'ThunderBird  - Native IVR',
    'ThunderBird - Media File Management',
    'Thunderbird - MFM Core',
    'ThunderBird - Omni Flow (Workflow Engine)',
    'ThunderBird - Transcription',
    'ThunderBird - Native Voice - Team-1',
    'ThunderBird - Native Voice - Team-2',
    'ThunderBird - Native Voice - Team-3',
    'ThunderBird - Voice Infra - Signal Wire and Abstraction Layer (SCPV)'
  )
  ORDER BY Name
" --json
```

## Step 2: Fetch all P0/P1/P2 bugs by product tag

**No date filter. No build filter.** Scope is all-time P0/P1/P2 bugs on these product tags.

```bash
sf data query --target-org gus --query "
  SELECT Id, Name, Subject__c, Status__c, Priority__c,
    Product_Tag__r.Name, CreatedDate, LastModifiedDate
  FROM ADM_Work__c
  WHERE Type__c = 'Bug'
    AND Priority__c IN ('P0', 'P1', 'P2')
    AND Product_Tag__c IN ('TAG_ID_1','TAG_ID_2','TAG_ID_N')
  ORDER BY CreatedDate
  LIMIT 2000
" --json
```

If result count hits 2000, paginate with `OFFSET`.

## Step 3: Weekly bucketing logic

For each bug, use **two date fields**:
- `opened_week` = ISO week of `CreatedDate` → increment `opened` count for that week
- `closed_week` = ISO week of `LastModifiedDate`, **only if current `Status__c` is in the closed set** → increment `closed` count for that week

```python
CLOSED_STATUSES = {
    'Fixed', 'Never Fix', 'Not a Bug', 'Not Reproducible', 'Closed',
    'Duplicate', 'Will Not Fix', 'Integrate', 'Pending Release',
    'Never', 'Not a bug'
}

def get_week(dt_str):
    from datetime import datetime
    dt = datetime.fromisoformat(dt_str.replace('+0000', '+00:00'))
    return dt.strftime('%Y-W%W')

for rec in recs:
    tag         = rec['Product_Tag__r']['Name']
    opened_week = get_week(rec['CreatedDate'])
    is_closed   = rec['Status__c'] in CLOSED_STATUSES
    closed_week = get_week(rec['LastModifiedDate']) if is_closed else None

    by_project[tag][opened_week]['opened'] += 1
    if closed_week:
        by_project[tag][closed_week]['closed'] += 1
```

## Step 4: Compute three series per project (for last 6 weeks)

Compute cumulative net open **starting from the beginning of all history**, not just the display window. This ensures the net open line reflects true backlog.

```python
def compute_series(tag_data, all_weeks, display_weeks):
    cum_opened = cum_closed = 0
    display_set = set(display_weeks)
    opened_s = []; closed_s = []; net_s = []
    for w in all_weeks:                          # iterate ALL weeks in order
        cum_opened += tag_data.get(w, {}).get('opened', 0)
        cum_closed += tag_data.get(w, {}).get('closed', 0)
        if w in display_set:                     # only output for display window
            opened_s.append(tag_data.get(w, {}).get('opened', 0))
            closed_s.append(tag_data.get(w, {}).get('closed', 0))
            net_s.append(max(0, cum_opened - cum_closed))
    return opened_s, closed_s, net_s
```

- `display_weeks` = last 6 ISO weeks from the data
- Always show **last 6 weeks** on the chart X-axis

## Step 5: Optional cross-check via release report

If the user provides a report URL like:
`https://gus.lightning.force.com/lightning/r/Report/00OEE000002qk412AA/view`

Extract the report ID and fetch it to validate PPM project names and program groupings:

```bash
sf api request rest --target-org gus \
  "/services/data/v64.0/analytics/reports/REPORT_ID" \
  --method GET
```

Use `groupingsDown.groupings` to extract program names and `factMap` rows to extract project names. This confirms scope but **do not use for bug counts** — query GUS directly.
