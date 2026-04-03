---
name: backlog
description: View and manage the content backlog
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

# Manage Content Backlog

View and manage the content production backlog. Control content priorities, add new items, and organize by topic cluster.

## Execution Flow

### 1. Argument Handling and Mode Selection

Determine execution mode based on arguments:

- **No arguments (default)**: Show top 10 pending items sorted by priority
- **--add**: Add a new content item to backlog (interactive)
- **--prioritize**: Recalculate priority scores for all backlog items
- **--cluster CLUSTER**: Filter and show all items in specified cluster
- **--top N**: Show top N items (where N is a number, e.g., --top 20)
- **--top N --cluster CLUSTER**: Show top N items from specified cluster

### 2. Default Mode: Show Top 10 Items

1. **Query Backlog**:
   - Load all items from SQLite `backlog` table where status = 'pending'
   - Sort by priority_score DESC (highest priority first)
   - Limit to 10 items (or top N if --top N specified)

2. **Prepare Display**:
   - For each item, prepare columns:
     - **Rank**: Sequential rank number (1, 2, 3...)
     - **Title/Keyword**: Item keyword or suggested title
     - **Priority Score**: Numeric score (0-100)
     - **Cluster**: Topic cluster name
     - **Type**: Content type (article, guide, tutorial, case_study, etc.)
     - **Effort**: Estimated effort in tokens (low/medium/high or numeric estimate)
     - **Source**: Where the item came from (discovery, manual add, keyword research, etc.)
     - **Added Date**: When item was added to backlog

3. **Format Output**:
   - Display as formatted table with clear columns
   - Use visual indicators for priority (e.g., ★★★★★ for high priority)
   - Color-code or badge difficulty levels if supported
   - Show summary line: "Showing X of Y pending items"

4. **Provide Actions Menu**:
   - Prompt user with options:
     - "Enter item rank to select and publish (e.g., 1, 2, 3)"
     - "or enter: /add, /prioritize, /cluster, /help"

### 3. Add Mode (--add)

1. **Interactive Form**:
   - Use AskUserQuestion to collect:
     - **Keyword** (required): Primary keyword/topic for content
     - **Content Type** (required): Select from predefined types:
       - article
       - guide
       - tutorial
       - how-to
       - case_study
       - comparison
       - tool_review
       - listicle
       - checklist
       - template
       - webinar
       - video_script
       - whitepaper
       - ebook
       - interview
     - **Cluster** (required): Select existing cluster or create new
       - Show list of existing clusters from discovery or manual setup
       - Option to type new cluster name
     - **Persona** (optional): Select target persona from personas.yaml
       - If not selected, use default/primary persona
     - **Priority Hint** (optional): Ask user to estimate priority
       - "High (urgent, high-value topic)"
       - "Medium (standard priority)"
       - "Low (nice-to-have, longer tail)"
     - **Description** (optional): Brief description of intended content
     - **Source Keywords** (optional): If from keyword research, list source keywords
     - **Target URL** (optional): If targeting a specific page type or update

2. **Calculate Initial Priority**:
   - If user provided priority hint: map to initial score
   - Load scoring.yaml to get weight factors
   - Calculate initial priority based on:
     - User hint (if provided)
     - Cluster size and momentum
     - Topic relevance (from brand strategy)
     - Keyword search volume (if available)
   - Default: medium priority (50) if no signals

3. **Create Backlog Entry**:
   - Generate unique backlog item ID
   - Record all fields:
     - keyword, content_type, cluster, persona_id
     - priority_score (calculated)
     - status (pending)
     - created_date (now)
     - created_by (system or user)
     - source (manual_add)
   - Insert into SQLite backlog table
   - Record estimated tokens based on content_type:
     - article: 2000-3000
     - guide: 3000-4000
     - tutorial: 2500-3500
     - case_study: 2500-3500
     - listicle: 1500-2500
     - tool_review: 2000-3000
     - etc.

4. **Confirmation and Next Steps**:
   - Show created item with assigned rank
   - Display when it would be published (based on current queue)
   - Offer options:
     - "Start publishing now (--from-backlog ID)"
     - "Add another item"
     - "Back to menu"

### 4. Prioritize Mode (--prioritize)

1. **Load Scoring Rules**:
   - Read scoring.yaml which contains priority weight factors
   - Example structure:
     ```yaml
     weights:
       keyword_search_volume: 0.25
       cluster_momentum: 0.20
       brand_alignment: 0.15
       user_hint: 0.20
       freshness_bonus: 0.10
       strategic_initiative: 0.10
     ```

2. **Fetch Input Signals**:
   - For each item in backlog (status != published, status != rejected):
     - Keyword search volume: Query Google Search Console (if MCP available) or estimate
     - Cluster momentum: Analyze recent content in cluster (publication rate, engagement)
     - Brand alignment: Score against strategy.yaml (0-100)
     - User hint: Use user_hint_priority if provided, otherwise neutral (50)
     - Freshness bonus: Recently added items get small boost
     - Strategic initiatives: Check against any active initiatives in config

3. **Recalculate Scores**:
   - For each item:
     - priority_score = SUM(signal * weight) across all factors
     - Normalize to 0-100 scale
     - Apply cluster grouping: items in same cluster should have relative proximity
   - Update SQLite backlog table with new priority_scores
   - Record recalculation timestamp

4. **Report Changes**:
   - Show items that moved up/down significantly
   - Display top 10 with new rankings
   - Identify any items that dropped to very low priority (offer to archive)
   - Ask: "Do you want to archive low-priority items (score < 20)?"

### 5. Cluster Filter Mode (--cluster CLUSTER)

1. **Query by Cluster**:
   - Load all items from backlog where cluster = specified CLUSTER and status != rejected
   - Sort by priority_score DESC
   - If no items found: report and suggest adding items

2. **Cluster Summary**:
   - Show cluster metadata:
     - Cluster name
     - Total items in cluster
     - Average priority score
     - Most recent publication date in cluster
     - Internal linking opportunities (related items)
   - Show item count breakdown by status (pending, published, in_progress)

3. **Display Items**:
   - Show all items in cluster with same formatting as default mode
   - Include internal link opportunities: "X other items in this cluster could link to this"
   - Highlight any orphaned items (published but not linked from other cluster items)

4. **Cluster Actions**:
   - Offer to:
     - Select item from cluster to publish
     - View cluster content map (text-based graph of internal links)
     - Bulk update priority for items in cluster
     - Identify gaps in cluster coverage

### 6. Top N Mode (--top N)

1. **Parse Argument**:
   - Extract N from argument (e.g., --top 20 → N=20)
   - Validate N is a positive integer (1-100)

2. **Query and Display**:
   - Load top N pending items
   - Display with same formatting as default mode
   - Show total backlog size: "Showing 20 of 157 pending items"

3. **Can Combine with --cluster**:
   - --top 20 --cluster "SEO" shows top 20 items in SEO cluster
   - Useful for viewing subset of backlog

## Display Format

Display backlog items in a formatted table:

```
Rank | Keyword              | Score | Cluster      | Type      | Effort | Source
-----|----------------------|-------|--------------|-----------|--------|-------------
  1  | advanced-seo-tactics | 92    | SEO Strategy | guide     | HIGH   | keyword_res
  2  | wordpress-security   | 89    | WordPress    | tutorial  | HIGH   | discovery
  3  | link-building-2025   | 85    | Link Build   | article   | MED    | manual_add
  4  | image-optimization   | 78    | SEO Strategy | guide     | MED    | keyword_res
  5  | caching-strategies   | 72    | Performance  | tutorial  | HIGH   | discovery
```

For each item, show:
- Clear rank number
- Full keyword/topic
- Priority score (0-100, color-coded: red <30, yellow 30-70, green >70)
- Cluster name (clickable to filter by cluster)
- Content type (e.g., article, guide, tutorial)
- Estimated effort (LOW/MEDIUM/HIGH) based on content_type
- Source (keyword_research, discovery, manual_add, strategic_initiative)

## Database Queries

Key SQL patterns used:

```sql
-- Get top 10 pending items
SELECT id, keyword, priority_score, cluster, content_type,
       estimated_tokens, source, created_date
FROM backlog
WHERE status = 'pending'
ORDER BY priority_score DESC
LIMIT 10

-- Get items by cluster
SELECT * FROM backlog
WHERE cluster = ? AND status != 'rejected'
ORDER BY priority_score DESC

-- Backlog summary stats
SELECT cluster, COUNT(*) as item_count, AVG(priority_score) as avg_priority
FROM backlog
WHERE status = 'pending'
GROUP BY cluster
```

## Success Criteria

- Backlog displayed correctly with accurate priority scores
- Items can be added and immediately appear in backlog
- Prioritization recalculates scores based on configured weights
- Cluster filtering works correctly
- Top N selection respects specified count
- User can easily select items to publish
- No errors reading from SQLite backlog table

## Error Handling

- If backlog is empty: show message and suggest `/discover` or `/add`
- If cluster doesn't exist: report and offer to create new cluster or list existing clusters
- If priority calculation fails: fall back to existing scores and report issue
- If invalid N provided to --top: prompt for valid integer
- If database connection fails: report and suggest `/init --repair`

## Output

Display formatted backlog table with selected items, summary statistics, and action menu. Provide clear guidance for selecting items for publication or managing backlog priorities.
