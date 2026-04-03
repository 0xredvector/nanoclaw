---
name: publish
description: Produce and publish the next content piece
allowed-tools: Read, Write, Edit, Bash(*), Grep, Glob, WebFetch, WebSearch
---

## Paths

- **Site root:** `/workspace/extra/site`
- **Plugin root:** `/workspace/extra/plugin`
- **Scripts:** `/workspace/extra/plugin/scripts`
- **Prompts:** `/workspace/extra/plugin/prompts`
- **Config templates:** `/workspace/extra/plugin/config-templates`
- **Database:** `/workspace/group/content-autopilot.db`
- **Reports:** `/workspace/group/reports`

# Publish Next Content Piece

Execute the full content production pipeline: brief generation, outline, draft, quality gates, review, and publication.

## Execution Flow

### 1. Select Content Item

Determine which content item to produce:

1. **No Arguments (Default)**:
   - Query SQLite backlog table: `SELECT * FROM backlog WHERE status = 'pending' ORDER BY priority_score DESC LIMIT 1`
   - Use the highest priority pending item

2. **--from-backlog ID**:
   - Query backlog by specified ID
   - Verify item status is pending or ready_for_review
   - Proceed with that item

3. **--keyword KEYWORD**:
   - Search backlog for items matching keyword
   - If multiple matches: ask user to select via AskUserQuestion
   - Use selected item

4. **Validation**:
   - Verify item has required fields: keyword, content_type, cluster
   - Verify item has assigned persona
   - If missing, ask user for clarification via AskUserQuestion

### 2. Load Configuration and Templates

1. **Read Configuration Files**:
   - Load config.yaml (site info, CMS type, strategy)
   - Load personas.yaml (target personas)
   - Load brand_voice.yaml (brand voice profile)
   - Load qa.yaml (quality gate rules)

2. **Load Prompt Templates**:
   - Load brief generator prompt: `${CLAUDE_PLUGIN_ROOT}/prompts/content/brief_generator.md`
   - Load outline generator prompt: `${CLAUDE_PLUGIN_ROOT}/prompts/content/outline_generator.md`
   - Load draft generator prompt: `${CLAUDE_PLUGIN_ROOT}/prompts/content/draft_generator.md`
   - Load revision prompt (if needed): `${CLAUDE_PLUGIN_ROOT}/prompts/content/revision.md`

### 3. Generate Brief

1. **Prepare Context**:
   - Extract persona profile from personas.yaml for the assigned persona
   - Extract brand voice profile from brand_voice.yaml
   - Extract content item: keyword, content_type, cluster
   - Load any reference content from the same cluster (for consistency)

2. **Render Brief Template**:
   - Use brief_generator.md prompt template
   - Substitute variables:
     - {{ keyword }} = content item keyword
     - {{ content_type }} = article/guide/tutorial/case_study/etc.
     - {{ persona }} = target persona description
     - {{ brand_voice }} = brand voice profile
     - {{ cluster }} = topic cluster name and related content
   - Execute prompt to generate brief (500-800 tokens)

3. **Store Brief**:
   - Save brief text to SQLite `content_generation` table with status='brief_generated'
   - Record tokens used and cost
   - Record generation timestamp

### 4. Generate Outline

1. **Prepare Context**:
   - Load the generated brief from previous step
   - Load supporting research (Search Console queries if available, related content)

2. **Render Outline Template**:
   - Use outline_generator.md prompt template
   - Substitute variables:
     - {{ brief }} = generated brief from step 3
     - {{ content_type }} = article type
     - {{ persona }} = target persona
     - {{ brand_voice }} = brand voice profile
   - Execute prompt to generate outline (800-1200 tokens)
   - Outline should include:
     - Main sections with H2 headings
     - 2-3 subsections per main section with H3 headings
     - Key points to cover in each section

3. **Store Outline**:
   - Save outline text to SQLite `content_generation` table with status='outline_generated'
   - Record tokens used and cost
   - Record generation timestamp

### 5. Generate Full Draft

1. **Prepare Context**:
   - Load brief and outline from previous steps
   - Load any research materials (web search results if needed)
   - Load recent content from same cluster as reference for style/structure

2. **Render Draft Template**:
   - Use draft_generator.md prompt template
   - Substitute variables:
     - {{ outline }} = generated outline from step 4
     - {{ brief }} = generated brief
     - {{ keyword }} = primary keyword
     - {{ brand_voice }} = brand voice profile
     - {{ cluster }} = topic cluster for cross-linking opportunities
   - Execute prompt to generate full draft (2000-4000 tokens depending on content type)
   - Draft should be complete, publishable content

3. **Store Draft**:
   - Save draft text to SQLite `content_generation` table with status='draft_generated'
   - Record tokens used and cost
   - Record generation timestamp
   - Calculate word count

### 6. Run Quality Gates

1. **Load QA Profile**:
   - Read qa.yaml for quality gate rules
   - Determine profile: solo, startup, or editorial

2. **Quality Checks** (vary by profile):

   **Solo Profile** (minimal checks):
   - Syntax: Check for obvious errors (unmatched brackets, formatting issues)
   - Length: Verify word count meets minimum (e.g., 500 words)
   - Keywords: Verify primary keyword appears in title, first paragraph, headings
   - Links: Check that at least 2-3 internal links are present
   - Readability: Simple checks (average sentence length, paragraph breaks)

   **Startup Profile** (medium checks):
   - All solo profile checks
   - Structure: Verify proper heading hierarchy (H1 > H2 > H3)
   - SEO: Verify meta description is present and length (150-160 chars)
   - Images: Verify at least one image is referenced (for illustration)
   - Uniqueness: Check not too similar to existing content
   - Tone: Basic check that brand voice characteristics are evident

   **Editorial Profile** (comprehensive checks):
   - All startup profile checks
   - Grammar: Check for grammatical errors (passive voice, common mistakes)
   - Style: Verify consistency with brand voice (tone, vocabulary, sentence structure)
   - Fact accuracy: Flag any claims that should be verified
   - Citation quality: Check that external links are credible
   - Originality: Detailed comparison with existing content
   - Requires human review before publication

3. **Quality Scoring**:
   - Assign numeric score to each check (0-100 scale)
   - Calculate overall quality score as weighted average
   - Record individual scores and overall score

4. **Store Results**:
   - Save quality gate results to SQLite `quality_gates` table
   - Record status: pass/fail/review_required
   - Record specific failures and warnings

### 7. Auto-Revision (If Applicable)

If quality gates fail on auto-fixable issues and profile allows revision:

1. **Identify Fixable Issues**:
   - Issues marked as auto_fixable=true in qa.yaml
   - Examples: keyword not in title, missing meta description, paragraph too long, etc.

2. **Attempt Revision** (max 2 attempts):
   - For each auto-fixable failure:
     - Load revision prompt from `${CLAUDE_PLUGIN_ROOT}/prompts/content/revision.md`
     - Specify the issue to fix
     - Execute revision prompt on draft
     - Update draft with revision
   - Re-run quality gates on revised draft
   - If passes or achieves >= 80 quality score: proceed to publication
   - If still fails: stop and escalate to manual review

3. **Track Revisions**:
   - Record number of revision attempts
   - Record what was fixed in each attempt
   - Store revised draft with status='draft_revised'

### 8. Link and Media Checks

1. **Link Validation**:
   - Extract all URLs from draft
   - For each URL:
     - Verify it is properly formatted
     - If internal link: verify target page exists (check against discovered content)
     - If external link: optional HTTP HEAD request to verify (skip if would slow down)
     - Mark broken links for review
   - Flag any missing links that should be present (cluster-related topics)

2. **Media Checks**:
   - Count image references in draft
   - If content_type=article: require minimum 1 image (preferably 2-3)
   - If images not provided: suggest adding (don't block)
   - Verify image alt text placeholders are present

3. **Store Results**:
   - Save link/media check results to SQLite
   - Record any broken links or missing media

### 9. Manual Review (If Required)

For editorial profile or if quality gates failed:

1. **Prepare Review Package**:
   - Compile full draft with:
     - Generated brief and outline
     - Quality gate results and scores
     - Any issues flagged for review
     - Proposed solutions (auto-fixes)

2. **Present to User**:
   - Use AskUserQuestion to present draft for approval
   - Show:
     - Full draft text
     - Quality score and breakdown
     - Any issues and recommended fixes
     - Options: Approve / Request Revisions / Reject

3. **Handle Response**:
   - Approve: Proceed to publication
   - Request Revisions: Ask for specific changes, update draft, re-run quality gates
   - Reject: Mark item as rejected in backlog, record reason

### 10. Publish

Publish the approved content to the configured CMS:

1. **Prepare Metadata**:
   - Extract from config: site domain, CMS type
   - Extract from item: keyword (as slug), content_type, cluster
   - Generate metadata:
     - Slug: URL-safe version of keyword
     - Meta description: First sentence of brief or generated description
     - Featured image: If available
     - Categories/Tags: From cluster and persona
     - Author: From config (if available)

2. **Publication Steps** (vary by CMS):

   **WordPress**:
   - Connect via REST API or XML-RPC (credentials from config)
   - Create post with title, content, excerpt
   - Set categories, tags, featured image
   - Set post status to published
   - Retrieve published URL

   **HubSpot**:
   - Use HubSpot API (credentials from config)
   - Create blog post object
   - Set title, content, meta description
   - Add to topic clusters
   - Publish

   **Static Site / Manual**:
   - Write markdown file to output directory
   - Filename: `[keyword]-[timestamp].md`
   - Include YAML frontmatter with metadata
   - Export instructions for manual upload

3. **Post-Publication**:
   - Verify publication was successful
   - Record published URL
   - Update backlog status to published
   - Store content record in SQLite `content` table with status=published

4. **Cleanup**:
   - Archive intermediate outputs (brief, outline) in SQLite
   - Update metrics table with resource usage

### 11. Argument Handling

**--dry-run**:
- Execute entire pipeline through step 8 (manual review)
- Do not execute step 10 (publication)
- Present final draft for user confirmation
- Show all resource costs and estimates
- Exit without persisting changes

**--from-backlog ID** and **--keyword KEYWORD**:
- See step 1 (selection logic)

## Success Criteria

- Content item selected successfully
- Brief, outline, and draft generated
- Quality gates evaluated (pass or identified issues)
- Auto-fixable issues corrected
- Manual review completed (if required)
- Content published to target CMS
- SQLite records updated with publication details
- No resource budget exceeded (in single run)

## Error Handling

- If backlog is empty: inform user and suggest running `/discover`
- If configuration missing required fields: ask user to fill via AskUserQuestion
- If prompt template not found: report error with path and recommend `/test --prompts`
- If CMS publication fails: rollback, record error, suggest manual publication
- If token budget exceeded during generation: pause and ask user for approval to continue
- If quality gates fail critically and no revision allowed: escalate to manual review with full details

## Output

Print progress through pipeline showing:
- Content item selected (keyword, type, persona)
- Brief, outline, and draft generated (token counts)
- Quality gate scores and any issues
- Any revisions attempted and results
- Final draft for review (if manual review needed)
- Publication status and URL (if published)
- Resource usage summary (tokens, cost, time)

For --dry-run: Show all above but clearly indicate no changes were persisted, with option to proceed with publication.
