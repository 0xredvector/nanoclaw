---
name: distribute
description: Distribute published content across channels
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

# Content Distribution

Generate distribution plans, social snippets, and repurposing ideas for published content.

## Execution Flow

### 1. Argument Handling

- **content_id**: Create distribution plan for a specific content piece
- **--plan**: Generate distribution plans for all recently published (undistributed) content
- **--status**: Show distribution status across all channels
- **--repurpose**: Suggest repurposing opportunities for top-performing content

### 2. Single Content Distribution (content_id)

1. Load content piece and performance data
2. Generate distribution plan via `DistributionManager.create_plan()`
3. For each enabled channel, generate social snippets using social-snippet prompt
4. Display plan:
   ```
   Distribution Plan — "FastAPI Complete Guide"
   ══════════════════════════════════════════════

   Channel    | Status   | Scheduled      | Snippet Preview
   ───────────┼──────────┼────────────────┼─────────────────
   Twitter    | ready    | Feb 24, 10am   | "FastAPI just made..."
   LinkedIn   | ready    | Feb 24, 11am   | "If you're building..."
   Newsletter | pending  | Next digest    | "This week's deep..."
   Reddit     | skipped  | —              | Low relevance score
   ```
5. Ask: "Should I finalize these snippets for distribution?"

### 3. Batch Plan (--plan)

1. Find all content with status = published AND no distribution plan
2. For each, generate plan with channel selection based on:
   - Content type (tutorials → Reddit/HN, thought leadership → LinkedIn)
   - Target audience alignment
   - Channel performance history
3. Display summary of all pending distributions
4. Ask: "Create plans for N pieces?"

### 4. Status Dashboard (--status)

1. Load all distribution records from DB
2. Show per-channel metrics:
   - Total posts per channel
   - Click-through rates (if available)
   - Best performing channel
   - Distribution frequency
3. Show content with no distribution plan

### 5. Repurpose Mode (--repurpose)

1. Identify top-performing content (by traffic, engagement)
2. Generate repurposing suggestions via `DistributionManager.suggest_repurposing()`:
   - Long article → Twitter thread
   - Tutorial → Video script outline
   - Comparison → Infographic data points
   - Guide → Email course series
3. Prioritize by effort vs potential reach
4. Ask: "Should I add repurposing tasks to the backlog?"

## Error Handling

- If no published content: suggest running /produce first
- If channel not configured: skip with note
- If social snippet generation fails: use title + URL as fallback

## Output

Display distribution plans and snippets. Save plans to database.
