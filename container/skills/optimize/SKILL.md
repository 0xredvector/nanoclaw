---
name: optimize
description: Identify and optimize underperforming content
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

# Content Optimization

Detect underperforming content and apply targeted optimizations to improve rankings and traffic.

## Execution Flow

### 1. Argument Handling

- **--scan**: Scan all published content for optimization opportunities
- **--queue**: Show the current optimization queue sorted by priority
- **--next**: Pick up the next highest-priority optimization and execute it
- **--report**: Show optimization impact report (completed optimizations and their results)
- **content_id**: Optimize a specific content piece by ID

### 2. Scan Mode (--scan)

1. Load all published content_pieces from SQLite
2. Load performance data from performance_snapshots (last 90 days)
3. Run `ContentOptimizer.scan_for_opportunities()` with 8 detection rules:
   - Decaying traffic (>20% decline) → content_refresh
   - Position 4-20 → keyword_expansion / title_rewrite
   - Low CTR (<2% with impressions) → title_rewrite + meta_update
   - Thin content (<800 words) → content_refresh
   - Outdated (>12 months) → content_refresh
   - Missing SEO elements → meta_update
   - Poor internal linking → internal_linking
   - Duplicate topics → merge recommendation
4. Save opportunities to optimization_items table
5. Display summary:
   - Total opportunities found by type
   - Top 10 highest priority items
6. Ask user: "Should I add these to the optimization queue?"

### 3. Queue Mode (--queue)

Display optimization queue:
```
Optimization Queue (N items)
═══════════════════════════

# | Title                          | Type            | Priority | Status
──┼────────────────────────────────┼─────────────────┼──────────┼────────
1 | Python Guide 2024              | content_refresh | 92       | queued
2 | Best IDE Comparison            | title_rewrite   | 85       | queued
3 | Docker Tutorial                | internal_linking| 78       | in_progress
```

### 4. Execute Optimization (--next or content_id)

1. Pick item from queue (highest priority or specified ID)
2. Load full content piece and performance data
3. Create optimization plan via `ContentOptimizer.create_optimization_plan()`
4. Based on optimization_type, use appropriate prompt template:
   - **title_rewrite**: Use `optimization/title-ab-test.md` → generate 3 variants, present to user
   - **meta_update**: Use `content/meta-description.md` → generate new meta description
   - **content_refresh**: Use `optimization/content-audit.md` → generate full audit with action items
   - **section_rewrite**: Use `optimization/rewrite-section.md` → rewrite weakest section
   - **internal_linking**: Use `optimization/internal-link-suggest.md` → suggest link placements
   - **keyword_expansion**: Analyze content for missing keyword variations, suggest additions
5. Present recommendations to user
6. If approved: apply changes (or provide CMS-ready content)
7. Mark optimization as completed with result data

### 5. Report Mode (--report)

Show optimization impact:
```
Optimization Impact Report
══════════════════════════

Completed This Month: 12
Avg Position Improvement: +3.2 positions
Avg CTR Improvement: +1.1%

Recent Completions:
# | Title                     | Type           | Impact
──┼───────────────────────────┼────────────────┼──────────
1 | Python Guide              | content_refresh| Position: 8→4, CTR: +2.1%
2 | IDE Comparison            | title_rewrite  | CTR: +1.5%
```

## Playbook Integration

This command interacts with playbooks:
- PB-003 (Content Decay Response) may auto-queue items
- PB-006 (Quick Win Capture) may auto-queue items
- PB-002 (Optimization vs New) controls the optimization/new content ratio

## Error Handling

- If no performance data: run scan based on content age and SEO analysis only
- If optimization fails: keep item in queue, log error, suggest manual review
- If MCP unavailable for publishing: save optimized content as draft

## Output

Print optimization results and next steps. Save detailed reports to `reports/optimization_[date].md`.
