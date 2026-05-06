# GUS Bug Burndown — HTML Line Chart

After collecting bug data, generate a self-contained HTML file using the **exact template below** and open it in the browser. Do not deviate from this template — it ensures a consistent UI for all users.

## Chart sizing — IMPORTANT

**Do NOT use the canvas `height` attribute alone** — Chart.js with `responsive: true` ignores it and scales based on container width, producing oversized charts.

**Always use `maintainAspectRatio: false` + a CSS wrapper div with an explicit `height`:**
- Combined chart wrapper: `height: 280px`
- Per-project chart wrappers: `height: 150px`

## Status classification

```python
CLOSED_STATUSES = {
    'Fixed', 'Never Fix', 'Not a Bug', 'Not Reproducible', 'Closed',
    'Duplicate', 'Will Not Fix', 'Integrate', 'Pending Release',
    'Never', 'Not a bug'
}
# Everything else = Open
```

## Full Python generation code

Use the Python sandbox (`mcp__plugin_aisuite_aisuite__python`) with this exact code:

```python
import json, re
from collections import defaultdict
from datetime import datetime

CLOSED_STATUSES = {'Fixed', 'Never Fix', 'Not a Bug', 'Not Reproducible', 'Closed',
                   'Duplicate', 'Will Not Fix', 'Integrate', 'Pending Release',
                   'Never', 'Not a bug'}

def get_week(dt_str):
    dt = datetime.fromisoformat(dt_str.replace('+0000', '+00:00'))
    return dt.strftime('%Y-W%W')

def safe_id(name):
    return re.sub(r'[^a-zA-Z0-9]', '_', name)

# --- load recs, by_project, combined, all_weeks, display_weeks from query step ---

week_labels = [w.replace('-W', ' W') for w in display_weeks]
COMBINED_H  = "280px"
PROJECT_H   = "150px"

def make_line_chart(canvas_id, opened, closed, net_open, wrapper_height):
    return f"""
      <div style="position:relative;height:{wrapper_height};">
        <canvas id="{canvas_id}"></canvas>
      </div>
      <script>
        (function() {{
          new Chart(document.getElementById('{canvas_id}'), {{
            type: 'line',
            data: {{
              labels: {json.dumps(week_labels)},
              datasets: [
                {{
                  label: 'Opened (new this week)',
                  data: {json.dumps(opened)},
                  borderColor: '#4e8cff', backgroundColor: 'rgba(78,140,255,0.07)',
                  fill: true, tension: 0.35, pointRadius: 4, pointHoverRadius: 6, borderWidth: 2.5
                }},
                {{
                  label: 'Closed (this week)',
                  data: {json.dumps(closed)},
                  borderColor: '#4bc08a', backgroundColor: 'rgba(75,192,138,0.07)',
                  fill: true, tension: 0.35, pointRadius: 4, pointHoverRadius: 6, borderWidth: 2.5
                }},
                {{
                  label: 'Net Open (cumulative)',
                  data: {json.dumps(net_open)},
                  borderColor: '#e84040', backgroundColor: 'rgba(232,64,64,0.06)',
                  fill: true, tension: 0.35, pointRadius: 4, pointHoverRadius: 6, borderWidth: 2.5
                }}
              ]
            }},
            options: {{
              responsive: true,
              maintainAspectRatio: false,
              interaction: {{ mode: 'index', intersect: false }},
              plugins: {{ legend: {{ position: 'top', labels: {{ boxWidth: 12, font: {{ size: 11 }} }} }} }},
              scales: {{
                x: {{ ticks: {{ font: {{ size: 10 }}, maxRotation: 30 }} }},
                y: {{ beginAtZero: true, ticks: {{ stepSize: 1, font: {{ size: 10 }} }} }}
              }}
            }}
          }});
        }})();
      </script>"""

combined_panel = f"""
  <div class="combined-panel">
    <div class="combined-header-bar">
      <div class="combined-title">All Projects Combined</div>
      <div class="panel-meta">
        <span class="badge p0">P0: {{total_p0}}</span>
        <span class="badge p1">P1: {{total_p1}}</span>
        <span class="badge p2">P2: {{total_p2}}</span>
        <span class="badge open-badge">Net Open: {{combined_net}}</span>
        <span class="badge total-badge">Total: {{total}}</span>
      </div>
    </div>
    <div class="combined-chart-wrap">
      {{make_line_chart('combined_all', c_o, c_c, c_n, COMBINED_H)}}
    </div>
  </div>"""

# build sections_html: one .program-section per program, .panels-grid inside with one .panel per tag

generated_at = datetime.now().strftime('%Y-%m-%d %H:%M')

html = f"""<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8"><meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>P0/P1/P2 Bug Burndown — Release {{release}} (THB)</title>
  <script src="https://cdn.jsdelivr.net/npm/chart.js@4.4.0/dist/chart.umd.min.js"></script>
  <style>
    *{{box-sizing:border-box;margin:0;padding:0;}}
    body{{font-family:-apple-system,BlinkMacSystemFont,'Segoe UI',Roboto,sans-serif;background:#f0f2f5;color:#1a1a2e;padding:24px;}}
    h1{{font-size:1.65rem;font-weight:800;color:#0d1117;}}
    .subtitle{{color:#777;font-size:0.88rem;margin-top:5px;margin-bottom:28px;}}
    .summary-bar{{display:flex;gap:12px;margin-bottom:30px;flex-wrap:wrap;}}
    .stat-card{{background:#fff;border-radius:12px;padding:14px 20px;box-shadow:0 1px 6px rgba(0,0,0,0.08);text-align:center;min-width:105px;}}
    .stat-card .val{{font-size:1.9rem;font-weight:800;line-height:1;}}
    .stat-card .lbl{{font-size:0.72rem;color:#999;margin-top:4px;text-transform:uppercase;letter-spacing:.05em;}}
    .v-gray{{color:#555;}}.v-red{{color:#e84040;}}.v-orange{{color:#ff9f40;}}.v-purple{{color:#9b59b6;}}
    .legend-box{{background:#fff;border-radius:10px;padding:12px 18px;margin-bottom:24px;
                 box-shadow:0 1px 5px rgba(0,0,0,0.07);display:flex;gap:24px;flex-wrap:wrap;}}
    .legend-item{{display:flex;align-items:center;gap:7px;font-size:0.82rem;color:#444;}}
    .legend-dot{{width:12px;height:12px;border-radius:50%;flex-shrink:0;}}
    .combined-panel{{background:#fff;border-radius:14px;margin-bottom:32px;
                     box-shadow:0 2px 10px rgba(0,0,0,0.09);overflow:hidden;}}
    .combined-header-bar{{display:flex;align-items:center;justify-content:space-between;
                          background:linear-gradient(135deg,#0d1117,#1a2744);
                          padding:14px 20px;gap:12px;flex-wrap:wrap;}}
    .combined-title{{font-size:1rem;font-weight:700;color:#ffffff;}}
    .combined-chart-wrap{{padding:18px 20px 16px;}}
    .program-section{{margin-bottom:30px;}}
    .section-header{{display:flex;align-items:center;justify-content:space-between;
                     background:#1a2744;color:#fff;border-radius:10px 10px 0 0;padding:11px 18px;}}
    .section-name{{font-weight:600;font-size:0.93rem;}}.section-stats{{font-size:0.8rem;opacity:.8;}}
    .panels-grid{{display:grid;grid-template-columns:repeat(auto-fill,minmax(440px,1fr));
                  gap:1px;background:#e2e6ea;border-radius:0 0 10px 10px;overflow:hidden;}}
    .panel{{background:#fff;padding:15px 16px 12px;}}
    .panel-header{{display:flex;align-items:flex-start;justify-content:space-between;
                   margin-bottom:10px;gap:8px;}}
    .panel-title{{font-size:0.8rem;font-weight:600;color:#333;line-height:1.35;}}
    .panel-meta{{display:flex;gap:5px;flex-shrink:0;flex-wrap:wrap;justify-content:flex-end;}}
    .badge{{font-size:0.68rem;padding:2px 7px;border-radius:20px;font-weight:600;}}
    .p0{{background:#f5e6ff;color:#6c3483;}}.p1{{background:#fde8e8;color:#c0392b;}}
    .p2{{background:#fef3e2;color:#d35400;}}.open-badge{{background:#fde8e8;color:#c0392b;}}
    .total-badge{{background:#eef2ff;color:#3b5bdb;}}
    .footer{{text-align:center;color:#bbb;font-size:0.76rem;
             margin-top:28px;padding-top:14px;border-top:1px solid #e0e0e0;}}
  </style>
</head>
<body>
  <h1>P0/P1/P2 Bug Burndown — Release {{release}} (THB)</h1>
  <div class="subtitle">Last 6 weeks &nbsp;·&nbsp; Generated: {{generated_at}}</div>
  <div class="summary-bar">
    <div class="stat-card"><div class="val v-gray">{{total}}</div><div class="lbl">Total Bugs</div></div>
    <div class="stat-card"><div class="val v-purple">{{total_p0}}</div><div class="lbl">P0</div></div>
    <div class="stat-card"><div class="val v-red">{{total_p1}}</div><div class="lbl">P1</div></div>
    <div class="stat-card"><div class="val v-orange">{{total_p2}}</div><div class="lbl">P2</div></div>
    <div class="stat-card"><div class="val v-red">{{combined_net}}</div><div class="lbl">Net Open</div></div>
    <div class="stat-card"><div class="val v-gray">{{len(by_project)}}</div><div class="lbl">Projects</div></div>
  </div>
  <div class="legend-box">
    <div class="legend-item"><div class="legend-dot" style="background:#4e8cff"></div>
      <b>Opened (new)</b> — bugs created that week (CreatedDate)</div>
    <div class="legend-item"><div class="legend-dot" style="background:#4bc08a"></div>
      <b>Closed</b> — bugs closed that week (LastModifiedDate, status = closed)</div>
    <div class="legend-item"><div class="legend-dot" style="background:#e84040"></div>
      <b>Net Open (cumulative)</b> — running total open bugs as of end of that week</div>
  </div>
  {{combined_panel}}
  {{sections_html}}
  <div class="footer">
    GUS data · THB product tags · P0, P1 &amp; P2 bugs (Type = Bug) · No build/date filter applied
  </div>
  <script>
    Chart.defaults.font.family="-apple-system,BlinkMacSystemFont,'Segoe UI',Roboto,sans-serif";
  </script>
</body>
</html>"""

# Write and open
with open(f'/tmp/bug_burndown_{{release}}.html', 'w') as f:
    f.write(html)
```

## Output file

- Write to `/tmp/bug_burndown_{release}.html`
- Copy to `/Users/amansaxena/Documents/Claude PM/bug_burndown_{release}.html`
- Open with Bash tool: `open /tmp/bug_burndown_{release}.html`
- Use Chart.js 4.4.0 from CDN: `https://cdn.jsdelivr.net/npm/chart.js@4.4.0/dist/chart.umd.min.js`

## Footer note

Always include in the chart footer:
> GUS data · THB product tags · P0, P1 & P2 bugs (Type = Bug) · No build/date filter applied
