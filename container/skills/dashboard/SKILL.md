---
name: dashboard
description: Generate visual dashboards and interactive UI views
allowed-tools: Read, Write, Bash(*), Grep, Glob
---

## Paths

- **Site root:** `/workspace/extra/site`
- **Plugin root:** `/workspace/extra/plugin`
- **Scripts:** `/workspace/extra/plugin/scripts`
- **Prompts:** `/workspace/extra/plugin/prompts`
- **Config templates:** `/workspace/extra/plugin/config-templates`
- **Database:** `/workspace/group/content-autopilot.db`
- **Reports:** `/workspace/group/reports`

# Dashboard & Visual Interface

Generate interactive visual interfaces for the content autopilot system.

## Execution Flow

### 1. Argument Handling

- **--html** (default): Generate full HTML dashboard (Chart.js) and save to reports/
- **--status**: Render StatusDashboard React artifact in conversation
- **--calendar**: Render CalendarView React artifact in conversation
- **--backlog**: Render BacklogBoard React artifact in conversation
- **--pipeline**: Render PipelineView React artifact in conversation
- **--analytics**: Render AnalyticsView React artifact in conversation

### 2. HTML Dashboard (--html)

1. Import `DashboardGenerator` from `scripts/dashboard_generator.py`
2. Run `generator.generate(report_type="full")`
3. Save HTML file to `reports/dashboard_YYYY-MM-DD.html`
4. Display file link to user

### 3. React Artifacts (--status, --calendar, etc.)

1. Import `UIDataBridge` from `scripts/ui_data_bridge.py`
2. Extract live data from SQLite using the appropriate method:
   - `bridge.extract_status_data()` for --status
   - `bridge.extract_calendar_data()` for --calendar
   - `bridge.extract_backlog_data()` for --backlog
   - `bridge.extract_pipeline_data()` for --pipeline
   - `bridge.extract_analytics_data()` for --analytics
3. Read the corresponding JSX template from `ui/artifacts/`
4. Inject extracted data by replacing the mock data constants
5. Save the patched JSX to the workspace folder
6. The artifact will render in the Cowork conversation

### 4. Artifact Component Map

| Flag | Component File | Data Bridge Method |
|------|---------------|-------------------|
| --status | ui/artifacts/StatusDashboard.jsx | extract_status_data() |
| --calendar | ui/artifacts/CalendarView.jsx | extract_calendar_data() |
| --backlog | ui/artifacts/BacklogBoard.jsx | extract_backlog_data() |
| --pipeline | ui/artifacts/PipelineView.jsx | extract_pipeline_data() |
| --analytics | ui/artifacts/AnalyticsView.jsx | extract_analytics_data() |

### 5. Data Flow

```
SQLite DB
    ↓
UIDataBridge (Python)
    ↓ extracts & formats
JSON data object
    ↓ injected into
JSX artifact template
    ↓ rendered by
Cowork React runtime
```

For the HTML dashboard:
```
SQLite DB
    ↓
DashboardGenerator (Python)
    ↓ extracts & renders
Self-contained HTML file (Chart.js + inline data)
    ↓
User opens in browser
```

## Error Handling

- If DB is empty: artifacts render with sample/mock data (built into JSX templates)
- If DB partially populated: bridge returns defaults for missing tables
- If artifact file missing: error message with available options
- If reports/ dir doesn't exist: create it

## Notes

- HTML dashboard is a **snapshot** — regenerate for fresh data
- React artifacts show **mock data by default** and can be enhanced with live data
- Both approaches read the same SQLite database
- HTML dashboard is sharable (email, Slack) without Cowork
- React artifacts are interactive within the Cowork conversation
