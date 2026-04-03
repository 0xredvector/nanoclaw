---
name: report
description: Generate content marketing performance reports
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

# Report Generation

Generate comprehensive content marketing reports for different time periods and audiences.

## Execution Flow

### 1. Argument Handling

- **--weekly** (default): Generate weekly report for the last 7 days
- **--monthly**: Generate monthly report for the last 30 days
- **--dashboard**: Show compact real-time status dashboard
- **--custom START END**: Generate report for a custom date range

### 2. Data Collection

For any report type, gather data from SQLite:

1. **Content Activity**:
   - Content published in period (from content_pieces WHERE published_at in range)
   - Content optimized in period (from optimization_items WHERE completed_at in range)
   - Backlog changes (items added, completed, archived)

2. **Performance Metrics** (from performance_snapshots, site_daily_metrics):
   - Total organic sessions (current period vs previous)
   - Total pageviews
   - Top 5 pages by traffic
   - Top 5 search queries
   - Traffic trend (growing/stable/declining)

3. **Pipeline Status** (from backlog_items):
   - Total items in backlog
   - Items by status (queued, in_progress, draft, review)
   - Estimated weeks of content

4. **Health Scores** (computed via HealthScorer):
   - Site health grade
   - Cluster health summary
   - Content needing attention

5. **Alerts** (from alerts):
   - Alerts fired in period
   - Alerts resolved
   - Active unresolved alerts

6. **Playbook Activity** (from playbook_firings):
   - Playbooks that fired
   - Actions taken

### 3. Weekly Report (--weekly)

Generate markdown report using `ReportGenerator.generate_weekly()`:

1. **Executive Summary** — AI-generated narrative of the week
2. **Content Published** — table with titles, keywords, QA scores
3. **Content Optimized** — improvements made and measured impact
4. **Performance Overview** — traffic metrics vs previous week with trends
5. **Pipeline Status** — what's in the queue and coming next
6. **Alerts & Actions** — notable events and responses
7. **Health Scores** — site and cluster grades
8. **Next Week Priorities** — top 3-5 focus areas

Save to `reports/weekly_[date].md`

### 4. Monthly Report (--monthly)

Generate expanded report using `ReportGenerator.generate_monthly()`:

All weekly sections plus:
- **Monthly Trend** — week-over-week progression chart
- **Competitive Landscape** — changes in competitor positions
- **Experiments** — A/B test results
- **Strategic Recommendations** — data-driven suggestions
- **Goals Review** — progress against targets from strategy.yaml

Save to `reports/monthly_[date].md`

### 5. Dashboard (--dashboard)

Compact real-time view using `ReportGenerator.generate_dashboard()`:

```
╔══════════════════════════════════════════════════╗
║  CONTENT AUTOPILOT — Dashboard                   ║
╠══════════════════════════════════════════════════╣
║                                                  ║
║  Site Health:  B+ (82/100)    ▁▃▅▆▇ (trending ↑)║
║                                                  ║
║  Pipeline:     3 published │ 2 in progress │ 15  ║
║                   this week │  in QA        │ queue║
║                                                  ║
║  Traffic:      12.4K sessions (+8.2% vs last wk) ║
║  Organic:      78% of total                       ║
║                                                  ║
║  Alerts:       0 critical │ 2 warning │ 1 info   ║
║                                                  ║
║  Next Up:      "FastAPI Complete Guide" (pri: 92) ║
╚══════════════════════════════════════════════════╝
```

### 6. Present Report

1. Display report summary in console
2. Save full report as markdown file
3. If monthly: ask "Should I share this report?" (future: email/Slack integration)

## Error Handling

- If no performance data: generate report with content activity only, note data gap
- If less than 1 week of data: generate partial report with available data
- If no content published: still generate report showing pipeline and health status

## Output

Print report summary to console. Save full markdown report to `reports/` directory.
