---
name: compete
description: Analyze competitors and identify content opportunities
allowed-tools: Read, Write, Bash(*), Grep, Glob, WebFetch, WebSearch
---

## Paths

- **Site root:** `/workspace/extra/site`
- **Plugin root:** `/workspace/extra/plugin`
- **Scripts:** `/workspace/extra/plugin/scripts`
- **Prompts:** `/workspace/extra/plugin/prompts`
- **Config templates:** `/workspace/extra/plugin/config-templates`
- **Database:** `/workspace/group/content-autopilot.db`
- **Reports:** `/workspace/group/reports`

# Competitive Analysis

Track competitors, analyze their content strategies, and identify opportunities.

## Execution Flow

### 1. Argument Handling

- **domain** (e.g., `/compete example.com`): Analyze a specific competitor
- **--all**: Analyze all competitors from competitors.yaml
- **--gaps**: Show content gaps where competitors rank but we don't
- **--report**: Generate comprehensive competitive intelligence report

### 2. Single Competitor Analysis (domain provided)

1. Load competitor config from competitors.yaml (or add new competitor)
2. Crawl competitor's sitemap to get their content inventory
3. For each page, extract: title, URL, estimated publish date, word count, topic
4. Profile the competitor:
   - Total content volume
   - Publishing frequency
   - Average article depth (word count)
   - Topic distribution
   - Content types (guides, comparisons, reviews, etc.)
5. Compare with our content:
   - Topic overlap: keywords both sites target
   - Our strengths: topics we cover better
   - Their strengths: topics they cover that we don't
6. Generate content gap opportunities
7. Save competitor profile to DB
8. Display results

### 3. Gap Analysis (--gaps)

1. Load all tracked competitors from DB
2. For each competitor, compare their keyword coverage vs ours
3. Aggregate gaps across all competitors
4. Prioritize by:
   - Number of competitors covering the topic (more = higher priority)
   - Estimated search volume potential
   - Alignment with our strategy pillars
5. Display:
   ```
   Content Gaps (competitor coverage we're missing)
   ═════════════════════════════════════════════════

   # | Topic/Keyword           | Competitors Covering | Priority | Action
   ──┼─────────────────────────┼──────────────────────┼──────────┼──────────
   1 | python async patterns   | 3 of 4 competitors   | 95       | New article
   2 | flask vs django 2025    | 2 of 4               | 88       | New comparison
   3 | python testing tools    | 2 of 4               | 82       | Expand existing
   ```

6. Ask user: "Should I add the top gaps to the backlog?"

### 4. All Competitors (--all)

1. Load all competitors from competitors.yaml
2. Profile each competitor (reuse cached data if <7 days old)
3. Generate comparative overview:
   - Content volume comparison (bar chart in ASCII)
   - Publishing frequency comparison
   - Topic coverage heatmap (which topics each competitor covers)
   - Our position: where we lead vs where we trail
4. Display leaderboard and strategic summary

### 5. Report Mode (--report)

Generate full competitive intelligence report (markdown):
- Executive summary
- Competitor profiles (each competitor)
- Content gap analysis
- SERP overlap analysis
- Strategic recommendations
- Action items

Save to `reports/competitive_[date].md`

## Competitive Response Integration

When new competitor content is detected:
- PB-005 (Competitive Response) evaluates urgency
- High-priority gaps auto-queue to backlog
- Optimization items created for topics where we have weaker content

## Error Handling

- If competitor site is unreachable: use cached data, note staleness
- If too many pages to crawl: limit to most recent 100 articles
- If no competitors configured: prompt user to add at least 1

## Output

Print competitive analysis results. Save detailed report if --report flag used.
