---
name: inventory
description: View and manage the content inventory with lifecycle states
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

# Content Inventory

View the live content inventory with lifecycle states, performance metrics, and health indicators.

## Execution Flow

### 1. Argument Handling

- **No argument or --all**: Show full inventory summary with counts by status
- **--published**: List all published content with performance metrics
- **--draft**: List content in pipeline (brief, outline, draft, review stages)
- **--decaying**: List published content showing traffic decline (requires performance data)
- **--orphaned**: List content with zero internal links pointing to it
- **--stats**: Show aggregate statistics and content health dashboard

### 2. Load Data

1. Connect to SQLite database
2. Query `content_pieces` table for all content
3. If performance data available, join with `performance_snapshots`
4. Load cluster assignments from `topic_clusters`

### 3. Display: Full Inventory (default / --all)

Show summary dashboard:

```
📊 Content Inventory Dashboard
═══════════════════════════════

Total Content: 47 pieces
├── Published:  32 (68%)
├── Draft:       8 (17%)
├── In Review:   3 (6%)
├── Brief:       2 (4%)
├── Archived:    2 (4%)

Content Types:
├── Articles:      28
├── Deep Guides:    8
├── Comparisons:    5
├── Reviews:        4
├── Listicles:      2

Health Overview:
├── Healthy (score 70+):    22 (69%)
├── Needs Attention (40-69): 7 (22%)
├── Critical (below 40):     3 (9%)
```

### 4. Display: Published Content (--published)

Table format:

```
# | Title                          | Type      | Published   | Words | Health | Cluster
──┼────────────────────────────────┼───────────┼─────────────┼───────┼────────┼──────────
1 | How to Set Up Python Venv      | article   | 2025-08-15  | 1,850 | 85/100 | Python Basics
2 | Python vs JavaScript 2025      | compare   | 2025-07-22  | 2,400 | 72/100 | Comparisons
...
```

If performance data available, add columns: Monthly Traffic, Position, Trend (↑↓→)

### 5. Display: Pipeline Content (--draft)

Show content currently being produced:

```
Pipeline Status:
═══════════════

Brief Stage (2):
  • "Ultimate Guide to FastAPI" — keyword: fastapi tutorial — priority: 87
  • "Python Automation Tools" — keyword: python automation — priority: 82

Outline Stage (1):
  • "Django vs Flask 2025" — keyword: django vs flask — priority: 79

Draft Stage (3):
  • "Web Scraping with Python" — 1,200/2,000 words — QA pending
  • "Python Testing Best Practices" — 2,800/3,000 words — QA pending
  • "Async Python Guide" — draft complete — in QA review

Review Stage (2):
  • "Python for Data Science" — QA score: 78/100 — revision needed
  • "API Design Patterns" — QA score: 91/100 — approved, ready to publish
```

### 6. Display: Decaying Content (--decaying)

List published content where traffic has declined:

```
⚠️  Decaying Content (traffic down >20% over 90 days)
═══════════════════════════════════════════════════════

# | Title                     | Published   | Peak Traffic | Current | Decline | Action
──┼───────────────────────────┼─────────────┼──────────────┼─────────┼─────────┼──────────
1 | Python 2024 Guide         | 2024-01-15  | 2,400/mo     | 800/mo  | -67%    | Update + retitle
2 | Best Python IDEs          | 2024-06-10  | 1,800/mo     | 1,200/mo| -33%    | Refresh content
```

If no performance data: "Performance data not available. Connect Search Console and Analytics to detect decaying content."

### 7. Display: Orphaned Content (--orphaned)

Content with no internal links pointing to it:

```
🔗 Orphaned Content (no internal links)
════════════════════════════════════════

These articles have no other pages linking to them. Adding internal links improves discoverability and SEO.

# | Title                          | Published   | Cluster
──┼────────────────────────────────┼─────────────┼──────────
1 | Advanced Python Decorators     | 2025-03-10  | Python Advanced
2 | Docker Compose Tutorial        | 2025-05-22  | DevOps
```

Suggest: "Would you like me to generate internal linking suggestions for these pages?"

### 8. Display: Statistics (--stats)

```
📈 Content Statistics
═════════════════════

Production Velocity:
├── This week:    2 articles published
├── This month:   7 articles published
├── Avg/month:    5.2 articles (last 6 months)
├── Time to publish: 4.2 days average

Word Count:
├── Total words:     87,400
├── Avg per article: 1,856 words
├── Longest:         4,200 words ("Complete Python Testing Guide")
├── Shortest:          850 words ("Python Quick Tips")

Cluster Coverage:
├── Python Basics:    12 articles (strong)
├── Web Development:   8 articles (moderate)
├── Data Science:      5 articles (building)
├── DevOps:            3 articles (weak — needs attention)
├── Machine Learning:  2 articles (emerging)

Quality:
├── Avg QA score:      76/100
├── Published w/o QA:  3 articles (legacy)
├── Revision rate:     23% (articles needing revision after first QA)
```

## Lifecycle States

Content flows through these states:
1. **idea** → in backlog, not yet started
2. **brief** → brief generated
3. **outline** → outline generated
4. **draft** → full draft written
5. **review** → in QA review
6. **revision** → failed QA, being revised
7. **approved** → passed QA, ready to publish
8. **published** → live on the site
9. **archived** → removed from active inventory

## Error Handling

- If SQLite has no data: "No content in inventory yet. Run `/publish` to produce your first article or `/discover` to import existing content."
- If performance data unavailable: show inventory without performance columns, note the gap

## Output

Print formatted inventory to console. Optionally save full export to `reports/inventory_[date].md`.
