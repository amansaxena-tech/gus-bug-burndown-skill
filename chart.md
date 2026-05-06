# GUS Bug Burndown — HTML Line Chart

After collecting bug data, generate a self-contained HTML file with Chart.js line charts and open it in the browser.

## Chart structure

- **Legend box** at top explaining all three lines
- **First panel**: combined all-projects line chart (all THB bugs aggregated)
- **Subsequent panels**: one line chart per product tag, grouped under their program section header
- **X-axis**: last 6 ISO weeks (e.g. `2026 W13` … `2026 W18`)
- **Three lines per chart**:
  - **Blue `#4e8cff` — Opened (new this week)**: bugs whose `CreatedDate` falls in that week
  - **Green `#4bc08a` — Closed (this week)**: bugs whose `LastModifiedDate` falls in that week AND current `Status__c` is in the closed set
  - **Red `#e84040` — Net Open (cumulative)**: running total = all-time opened minus all-time closed as of end of that week. Computed across full history, not just the display window.

## Status classification

```python
CLOSED_STATUSES = {
    'Fixed', 'Never Fix', 'Not a Bug', 'Not Reproducible', 'Closed',
    'Duplicate', 'Will Not Fix', 'Integrate', 'Pending Release',
    'Never', 'Not a bug'
}
# Everything else (New, Triaged, In Progress, Ready for Review, QA In Progress, Waiting) = Open
```

## Chart sizing — IMPORTANT

**Do NOT use the canvas `height` attribute alone** — Chart.js with `responsive: true` ignores it and scales based on container width, producing oversized charts.

**Always use `maintainAspectRatio: false` + a CSS wrapper div with an explicit `height`:**

- Combined chart wrapper: `height: 280px`
- Per-project chart wrappers: `height: 150px`
- This makes the combined chart just under 2× the height of project charts, as required.

## Chart.js template

```python
COMBINED_H = "280px"
PROJECT_H  = "150px"

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
```

- `week_labels` = display_weeks with `-W` replaced by ` W`

## Combined panel header

Use a dedicated `.combined-header-bar` div (not `.panel-header`) with a dark gradient background. The title must use `color: #ffffff` so it's visible against the dark background:

```html
<div class="combined-header-bar">
  <div class="combined-title">All Projects Combined</div>  <!-- white text -->
  <div class="panel-meta">...</div>
</div>
```

CSS:
```css
.combined-header-bar { display:flex; align-items:center; justify-content:space-between;
                       background:linear-gradient(135deg,#0d1117,#1a2744); padding:14px 20px; }
.combined-title { font-size:1rem; font-weight:700; color:#ffffff; }
```

## Badge layout per panel

Each panel header shows:
- `Net Open: N` (red badge) — net open count as of latest week
- `Total: N` (blue badge) — all-time total bugs for that product tag

Combined panel header additionally shows:
- `P0: N` (purple badge), `P1: N` (red badge), `P2: N` (orange badge)

Program section headers show:
- `Net Open: N` — sum of net open across all projects in program

## Output file

- Write to `/tmp/bug_burndown_{release}.html`
- Copy to `/Users/amansaxena/Documents/Claude PM/bug_burndown_{release}.html`
- Open with Bash tool: `open /tmp/bug_burndown_{release}.html`
- Use Chart.js 4.4.0 from CDN: `https://cdn.jsdelivr.net/npm/chart.js@4.4.0/dist/chart.umd.min.js`
- Keep HTML fully self-contained

## Footer note

Always include in the chart footer:
> GUS data · THB product tags · P0, P1 & P2 bugs only (Type = Bug) · No build/date filter applied
