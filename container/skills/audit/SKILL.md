---
name: audit
description: Run comprehensive site and content audits
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

# Site & Content Audit

Run comprehensive audits across all aspects of the content operation.

## Execution Flow

### 1. Argument Handling

- **--full** (default): Run all audit modules and generate comprehensive report
- **--seo**: Technical SEO audit only
- **--health**: Content and site health scoring
- **--eol**: End-of-life content scan
- **--experiments**: Review active experiments and results
- **--decisions**: Audit autonomous decision quality

### 2. Full Audit (--full)

Run all modules sequentially:
1. SEO Audit → technical issues, meta tags, structure
2. Health Scoring → site, cluster, and content grades
3. Performance Review → traffic trends, top/bottom performers
4. EOL Scan → content candidates for retirement
5. Optimization Queue → pending improvements
6. Calendar Status → upcoming schedule health
7. Budget Check → usage vs limits

Generate unified report saved to `reports/audit_[date].md`

### 3. SEO Audit (--seo)

1. Run `SEOAuditor.full_audit()` on all published content
2. Display:
   - Overall SEO score (A-F)
   - Issues by severity (critical, warning, info)
   - Top 5 critical issues to fix
   - Quick wins (easy fixes with high impact)
3. Compare with previous audit if available

### 4. Health Audit (--health)

1. Run `HealthScorer.score_site()` with all content and clusters
2. Display:
   - Site health grade and trend
   - Cluster health breakdown
   - Bottom 10 content pieces by health score
   - Improvement recommendations
3. Highlight changes since last audit

### 5. EOL Audit (--eol)

1. Run `EOLManager.scan_for_eol()`
2. Display:
   - Content approaching end-of-life
   - Merge candidates (duplicates/overlap)
   - Cannibalization issues
   - Redirect health check
3. Ask: "Should I create EOL proposals for the top candidates?"

### 6. Experiment Review (--experiments)

1. Load active and recently completed experiments
2. Display:
   - Running experiments with current data
   - Completed experiments with results
   - Early stopping recommendations
   - Statistical significance status
3. Suggest new experiments based on data

### 7. Decision Audit (--decisions)

1. Run `DecisionLogger.get_decision_quality()`
2. Display:
   - Decision accuracy rate
   - Human override frequency
   - Categories with most overrides
   - Confidence calibration (was high confidence = high accuracy?)
3. Recommend guardrail adjustments

## Error Handling

- If module unavailable: skip and note in report
- If insufficient data: run partial audit with available data
- If previous audit not found: skip trend comparison

## Output

Print audit results to console. Save full report to `reports/audit_[date].md`.
