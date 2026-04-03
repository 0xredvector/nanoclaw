---
name: research
description: Discover and evaluate keywords for content opportunities
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

# Keyword Research

Discover, evaluate, and score keywords for content production opportunities.

## Execution Flow

### 1. Argument Handling

- **Seed keywords provided** (e.g., `/research python automation, web scraping`): Use these as seed keywords for discovery
- **--from-strategy**: Extract seed keywords from strategy.yaml pillars and clusters
- **--quick-wins**: Analyze existing content for quick-win optimization opportunities (requires performance data)
- **--report**: Show keyword research report from the latest run (no new research)

### 2. Load Configuration

1. Read `config/strategy.yaml` for content pillars, clusters, and target audience
2. Read `config/scoring.yaml` for keyword scoring weights
3. Read `config/competitors.yaml` for competitor domains
4. Read `config/thresholds.yaml` for minimum scores and limits
5. If `--from-strategy`: extract all pillar names and cluster keywords as seeds

### 3. Keyword Discovery

For each seed keyword:

1. **Generate expanded keywords** using modifier patterns:
   - Question modifiers: "how to [seed]", "what is [seed]", "why [seed]"
   - Comparison modifiers: "[seed] vs", "[seed] alternative"
   - Intent modifiers: "[seed] tutorial", "[seed] guide", "[seed] tool", "[seed] review"
   - Long-tail expansions: combine seed with related terms from strategy
   - Year modifiers: "[seed] 2025", "[seed] 2026"

2. **Use web search** to validate keywords:
   - Search for each seed keyword via WebSearch
   - Analyze search results for:
     - Actual competition (who ranks on page 1)
     - Related searches suggested
     - Content types ranking (guides, lists, tools, etc.)
   - Extract additional keyword ideas from search suggestions

3. **Classify intent** for each keyword:
   - Informational: "how", "what", "why", "guide", "tutorial"
   - Commercial: "best", "top", "review", "compare", "vs"
   - Transactional: "buy", "price", "download", "free trial"
   - Navigational: brand-specific terms

### 4. Keyword Evaluation

For each discovered keyword, calculate scores:

1. **Relevance Score** (0-100): Alignment with strategy pillars
   - High overlap with pillar keywords → higher score
   - Matches target audience intent → bonus
   - Aligns with existing clusters → bonus

2. **Competition Estimate** (0-100):
   - More words in keyword → likely less competitive
   - Commercial modifiers → higher competition
   - Presence of major brands in SERPs → higher competition

3. **Opportunity Score** (0-100): Composite score
   - 40% relevance + 30% (100 - competition) + 30% intent value
   - Intent values: informational=60, commercial=90, transactional=80, navigational=30

4. **Quick Win Detection**:
   - Check if we already have content targeting this keyword
   - If existing content ranks positions 4-20 → quick win opportunity
   - Flag cannibalization risks (multiple pages targeting same keyword)

### 5. Store Results

1. Save all evaluated keywords to SQLite `keywords` table using `scripts/keyword_researcher.py`
2. Group keywords by cluster and by intent
3. Generate keyword research report

### 6. Present Results

Display a summary with:

- **Total keywords discovered**: X from Y seeds
- **By intent**: Informational (N), Commercial (N), Transactional (N), Navigational (N)
- **By cluster**: Show distribution across content pillars
- **Top 15 opportunities** (sorted by opportunity score):
  - Keyword | Intent | Relevance | Competition | Opportunity | Cluster
- **Quick wins found**: N opportunities to optimize existing content
- **Cannibalization warnings**: N keywords with competing pages

Ask user: "Should I add the top opportunities to the backlog? [Yes, top 10 / Yes, top 20 / Yes, all / No, let me review first]"

If approved, feed selected keywords to the backlog via `/backlog --add`.

### 7. Chain to Backlog (if approved)

- Create backlog items for approved keywords
- Auto-assign content type based on intent:
  - Informational + long topic → deep_guide
  - Informational + moderate → article
  - Commercial → comparison or review
  - Transactional → landing_page
- Calculate initial priority scores
- Print: "Added N items to backlog. Run `/backlog` to view the pipeline."

## Quick Wins Mode (--quick-wins)

1. Load all existing content_pieces from SQLite
2. Load performance data (if available from Search Console/Analytics)
3. Identify articles that:
   - Rank positions 4-20 (close to page 1)
   - Have high impressions but low CTR
   - Haven't been updated in 6+ months
   - Are missing key SEO elements
4. Score and rank optimization opportunities
5. Present list with: URL | Current Position | Keyword | Recommendation | Priority

## Error Handling

- If no strategy.yaml found: ask user for seed keywords manually
- If web search fails: continue with pattern-based expansion only
- If DB write fails: display results to user and suggest running `/test --connections`

## Output

Print formatted research report and optionally save to `reports/keyword_research_[date].md`.
