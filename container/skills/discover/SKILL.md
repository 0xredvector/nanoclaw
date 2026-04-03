---
name: discover
description: Analyze an existing site to inventory content and infer strategy
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

# Discover and Analyze Existing Site

Run brownfield discovery on the configured site to inventory existing content and infer content strategy, personas, and brand voice.

## Execution Flow

### 1. Argument Handling

- **No argument or --refresh**: Run full discovery pipeline
- **--tech-only**: Execute only steps 1-3 (crawl and CMS detection), skip content analysis
- **--inventory-only**: Execute only steps 1-2 (crawl and metadata extraction), skip CMS detection and analysis
- **--report**: Show the results of the last completed discovery run from SQLite (do not re-run)

### 2. Crawl Sitemap and Content Inventory

1. **Locate Sitemap**:
   - Read site domain from `config.yaml`
   - Try multiple locations in order:
     - `https://[domain]/sitemap.xml`
     - `https://[domain]/sitemap_index.xml`
     - `https://[domain]/sitemap-news.xml`
     - `https://[domain]/sitemap-pages.xml`
     - Use WebFetch to retrieve each candidate
   - Parse XML and extract all URLs

2. **For Each URL in Sitemap**:
   - Use WebFetch to retrieve the page
   - Extract and record:
     - Page title (from `<title>` tag)
     - Publication/update date (from meta tags: og:published_time, article:published_time, or last-modified)
     - Main heading (first `<h1>` tag)
     - Secondary headings (all `<h2>` tags)
     - Word count (count of visible text)
     - Meta description
     - Meta keywords
     - Open Graph tags (og:type, og:image, etc.)
     - All internal and external links
     - Image count and alt text
     - Any custom author/byline metadata
   - Store all extracted metadata in SQLite `discovery_content` table

3. **Handle Errors**:
   - If sitemap not found, attempt recursive crawl from homepage:
     - Start with domain homepage
     - Extract all links
     - Follow internal links up to 100 pages (respecting robots.txt)
     - Timeout individual page fetches after 10 seconds
   - Skip pages that fail to fetch, log failures in discovery_errors table

### 3. Detect CMS and Technology Stack

1. **Analyze HTTP Headers**:
   - Check for Server header (e.g., Apache, nginx, IIS)
   - Check for X-Powered-By or CMS-specific headers
   - Check for cookies or query parameters that indicate CMS

2. **Analyze HTML Patterns**:
   - Look for CMS-specific patterns in HTML:
     - WordPress: wp-content, wp-includes, wp-json
     - HubSpot: hubspot, hs-script-loader
     - Contentful: _links.asset patterns in page
     - Webflow: webflow specific classes
     - Statically generated: absence of CMS patterns
   - Check for build fingerprints or asset paths
   - Analyze link structures and URL patterns

3. **Record CMS Detection**:
   - Detected CMS type
   - Confidence score (0-100)
   - List of evidence points
   - Store in SQLite `discovery_tech` table

### 4. Fetch Search Console and Analytics Data (If MCP Available)

If configured MCP connectors are available:

1. **Search Console Integration**:
   - Query last 90 days of search data
   - Extract:
     - Top 50 queries by impressions
     - Average CTR and position
     - Geographic distribution
     - Device breakdown
   - Identify content gaps: queries with impressions but low CTR
   - Store in SQLite `discovery_search_console` table

2. **Analytics Integration**:
   - Query last 90 days of session data
   - Extract:
     - Top 50 pages by sessions
     - Average session duration per page
     - Bounce rate distribution
     - Entry/exit page analysis
     - Traffic sources (organic, direct, referral, etc.)
   - Store in SQLite `discovery_analytics` table

3. **Data Correlation**:
   - Match Search Console queries to high-performing pages
   - Identify underperforming pages (low traffic but published long ago)
   - Identify traffic trends (growing, stable, declining)

### 5. Cluster Articles by Topic

1. **Content Clustering**:
   - Load all discovered content from discovery_content table
   - Use simple keyword-based clustering:
     - Extract keywords from titles, headings, and meta descriptions
     - Calculate TF-IDF scores for each content piece
     - Group content by similar keywords and topics
     - Use threshold-based clustering (e.g., cosine similarity > 0.6)
   - Identify clusters with:
     - Cluster name/topic
     - Number of articles
     - Average quality (word count, structure completeness)
     - Date range (oldest to newest)
     - Internal linking patterns

2. **Gap Analysis**:
   - Identify high-traffic topics (from Analytics if available) with few content pieces
   - Identify low-traffic topics with many content pieces
   - Record potential opportunities and risks

3. **Store Results**:
   - Save cluster assignments in SQLite `discovery_clusters` table
   - Record cluster metadata and analysis

### 6. Infer Brand Voice

1. **Sample Content**:
   - Select top 15 articles by traffic/recency (or random if no traffic data)
   - Load their full text using WebFetch

2. **Analyze Voice Characteristics**:
   - Sentence length: calculate average sentence length (longer = formal, shorter = casual)
   - Use of first-person pronouns (we, us) vs. third-person or passive voice
   - Use of contractions (can't vs. cannot)
   - Exclamation point frequency
   - Question frequency
   - Use of industry jargon vs. plain language
   - Tone indicators: authoritative, conversational, friendly, technical, etc.

3. **Generate Voice Profile**:
   - Record inferred characteristics with confidence scores
   - Generate text description of brand voice
   - Store in SQLite `discovery_brand_voice` table

### 7. Infer Target Personas

1. **Content Analysis**:
   - Analyze article titles, headings, and content topics
   - Look for pattern indicators:
     - Beginner-level explanations → beginner persona
     - Advanced technical depth → advanced/expert persona
     - Business/ROI focus → decision-maker persona
     - How-to and tutorials → practitioner persona
     - Case studies and social proof → buying-stage persona

2. **Search Query Analysis** (if available):
   - Analyze top search queries that lead to content
   - Question-based queries ("How to...") → learning-focused persona
   - Problem-based queries ("Fix...") → problem-solver persona
   - Product research queries → buyer persona

3. **Generate Persona Profiles**:
   - For each identified persona:
     - Name/title
     - Goals
     - Pain points (inferred from content)
     - Content preferences (types of articles they engage with)
     - Experience level
     - Confidence score
   - Store in SQLite `discovery_personas` table

### 8. Generate Discovery Report

1. **Compile Findings**:
   - Create comprehensive discovery report with sections:
     - **Site Overview**: Domain, CMS type, total content pieces, date range
     - **Technology**: Detected tech stack, confidence score
     - **Content Inventory**: Total pieces, by cluster, by content type, date distribution
     - **Search and Analytics**: Top queries, top pages, traffic trends (if available)
     - **Content Clusters**: Identified topics, gaps, opportunities
     - **Brand Voice**: Inferred characteristics with samples
     - **Personas**: Identified personas with confidence scores
     - **Recommendations**: Proposed next steps

2. **Generate Proposals**:
   - For each inferred attribute (CMS, personas, brand voice):
     - Present detected value
     - Show confidence score
     - Provide evidence/samples
     - Offer option to accept/edit/reject

3. **Store Report**:
   - Save complete report in SQLite `discovery_reports` table
   - Save as markdown file: `discovery_report_[timestamp].md`

### 9. Present Proposals to User

Use AskUserQuestion to present each category of findings:

1. **Technology**:
   - "I detected the following CMS: [detected]. Confidence: [score]%. Is this correct? [Accept/Edit/Reject]"

2. **Brand Voice**:
   - Present inferred voice profile with sample quotes
   - Ask: "Does this match your brand voice? [Accept/Edit/Reject]"

3. **Personas**:
   - Present each identified persona with description
   - Ask: "Are these the right personas for your business? [Accept/Edit Add More/Remove]"

4. **Content Clusters**:
   - Present identified topic clusters
   - Ask: "Are these clusters complete? Any missing topics? [Accept/Edit]"

5. **Confirmation**:
   - Show summary of accepted configurations
   - Ask: "Should I save these to configuration files?"
   - If yes: write approved values to config.yaml, personas.yaml, brand_voice.yaml

### 10. Save Discoveries to Configuration

For any user-approved proposals:
- Update config.yaml with CMS type and discovered site metadata
- Create or update personas.yaml with identified personas
- Create or update brand_voice.yaml with inferred brand voice
- Record discovery metadata (timestamp, confidence scores) in system_config table
- Create backlog entries for any identified content gaps or opportunities

## Success Criteria

- Successfully crawled and analyzed site content
- CMS type detected (or recorded as unknown)
- Content inventory stored in database
- Brand voice and personas inferred and presented for approval
- Discovery report generated and saved
- User-approved configurations saved to YAML files
- No critical errors in data collection

## Error Handling

- If sitemap not found, fall back to recursive crawl
- If page fetch fails, skip and continue with others (record in errors table)
- If MCP connections unavailable, continue without Search Console/Analytics data (note as degraded)
- If clustering fails, use simpler keyword-based grouping
- If inference confidence is very low, present with caveat and ask for manual input

## Output

Print discovery summary with:
- Content inventory statistics
- Detected CMS type and confidence
- Identified clusters and topics
- Inferred brand voice summary
- Identified personas
- Prompts for user approval of key findings
- Recommendation for next action (configure in /init or proceed to /publish)
