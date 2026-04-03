---
name: site-status
description: Show current state of the content autopilot agent
allowed-tools: Read, Bash(*), Grep, Glob
---

## Paths

- **Site root:** `/workspace/extra/site`
- **Plugin root:** `/workspace/extra/plugin`
- **Scripts:** `/workspace/extra/plugin/scripts`
- **Prompts:** `/workspace/extra/plugin/prompts`
- **Config templates:** `/workspace/extra/plugin/config-templates`
- **Database:** `/workspace/group/content-autopilot.db`
- **Reports:** `/workspace/group/reports`

# Show Content Autopilot Status

Display the current operational state and health metrics of the content autopilot system.

## Execution Flow

### 1. Read Configuration and Database

1. Load `config.yaml` to get site configuration
2. Connect to the SQLite database
3. Query the following tables:
   - `system_config` - for agent phase and settings
   - `content` - for content health metrics
   - `alerts` - for active alerts
   - `backlog` - for workflow status
   - `metrics` - for resource usage and budget tracking
   - `workflow_runs` - for recent workflow status

### 2. Default Output (No Arguments)

Display a status summary with:
- **Agent Phase**: Current operational phase (initializing, discovering, producing, publishing, monitoring)
- **Site Health Score**: Overall health percentage (0-100) derived from:
  - Content freshness (last update dates)
  - Broken links count
  - SEO completeness percentage
  - Average content quality score
- **Content Health Average**: Mean quality score across all published content
- **Active Alerts Count**: Number of active alerts (non-dismissed, non-resolved)
- **DLQ Pending Count**: Dead Letter Queue items awaiting manual review
- **Recent Workflow Status**: Last 5 workflow executions with:
  - Workflow type (brief_generator, outline_generator, draft_generator, publish, etc.)
  - Execution timestamp
  - Result status (success/failure/manual_review)
  - Duration
- **Daily Spend vs Budget**: Current day's token/cost spend vs. configured daily budget
- **Next Scheduled Workflows**: Next 3 scheduled workflow executions with timestamps

### 3. Argument Handling

**--dashboard**: Display a rich visual dashboard format:
- Use ASCII art or formatted tables for visual hierarchy
- Show gauge-style indicators for health scores (e.g., ████████░░ 82%)
- Include trend indicators (↑ up, ↓ down, → stable) for key metrics
- Highlight alerts in red, warnings in yellow, info in default
- Show a timeline of recent workflow executions
- Include recommended actions based on current state

**--metrics**: Display detailed, exportable metrics:
- All metrics from default output, in structured format
- Extended metrics including:
  - Total content pieces published (all-time and this month)
  - Average tokens per publish
  - Total cost this month vs. monthly budget
  - Content velocity (pieces per day)
  - Quality trend (average score over last 7 days)
  - Backlog size and priority distribution
  - MCP connector uptime and health
- Output in JSON or CSV format suitable for export
- Include 90-day trend data if available

**--alerts**: Display active alerts with details:
- List all non-dismissed, non-resolved alerts
- For each alert show:
  - Alert ID
  - Severity (critical, warning, info)
  - Type (budget_exceeded, quality_low, link_broken, no_recent_content, etc.)
  - First occurrence timestamp
  - Last occurrence timestamp
  - Occurrence count
  - Description/details
  - Recommended action
- Sort by severity (critical first) then by recency
- Show alert dismissal/resolution options

### 4. Database Queries

Execute the following SQL queries to gather data:

```sql
-- Health score calculation
SELECT AVG(quality_score) as avg_quality,
       COUNT(*) as total_content,
       MAX(published_date) as latest_publish
FROM content WHERE status = 'published'

-- Active alerts
SELECT COUNT(*) as alert_count FROM alerts
WHERE dismissed = 0 AND resolved = 0

-- DLQ pending
SELECT COUNT(*) as dlq_count FROM backlog
WHERE status = 'dlq_pending'

-- Recent workflows
SELECT * FROM workflow_runs
ORDER BY executed_at DESC LIMIT 5

-- Budget tracking
SELECT SUM(token_cost) as daily_spend,
       SUM(usd_cost) as daily_usd_cost
FROM metrics WHERE date = CURRENT_DATE
```

### 5. Output Formatting

Default and --dashboard modes:
- Use human-readable formatting
- Group related metrics together
- Highlight critical issues
- Include timestamps in local time
- Show confidence/reliability indicators for derived metrics

--metrics mode:
- Provide JSON or CSV output option
- Include timestamp for data freshness
- Include schema/data dictionary for export consumption

--alerts mode:
- Number each alert
- Use color coding if terminal supports it
- Show clear remediation steps

## Success Criteria

- Successfully read configuration and database
- Display current operational state accurately
- No errors or missing data (flag if unable to read certain metrics)
- Output is human-readable and actionable

## Error Handling

- If database is corrupted or missing, display diagnostic message and recommend `/init --repair`
- If configuration is invalid, flag specific issues and recommend `/init --reconfigure`
- If certain metrics are unavailable, mark as "N/A" with explanation
- If MCP connections fail, note as degraded capability but continue with other metrics

## Output

Display selected information based on argument flag, with timestamps showing data freshness and recommended next actions based on current state.
