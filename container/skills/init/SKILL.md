---
name: init
description: Initialize or reconfigure the content autopilot agent
allowed-tools: Read, Write, Edit, Bash(*), Glob, Grep, WebFetch, WebSearch
---

## Paths

- **Site root:** `/workspace/extra/site`
- **Plugin root:** `/workspace/extra/plugin`
- **Scripts:** `/workspace/extra/plugin/scripts`
- **Prompts:** `/workspace/extra/plugin/prompts`
- **Config templates:** `/workspace/extra/plugin/config-templates`
- **Database:** `/workspace/group/content-autopilot.db`
- **Reports:** `/workspace/group/reports`

# Initialize or Reconfigure Content Autopilot

Initialize the content autopilot agent for a new site or reconfigure an existing installation.

## Execution Flow

### 1. Detect Installation State

Read the configuration files and database to determine if this is a greenfield (new) or brownfield (existing) installation:
- Check for existence of `${CLAUDE_PLUGIN_ROOT}/config.yaml`
- Check for existence of the SQLite database
- If both exist, treat as brownfield; otherwise treat as greenfield

### 2. Greenfield Path (New Site)

For new installations, run the initialization wizard:

1. **Ask for Core Configuration** using AskUserQuestion:
   - Domain (e.g., example.com)
   - CMS type (WordPress, HubSpot, Contentful, Static, Other)
   - Content strategy type (Blog, Product Docs, SaaS Marketing, News, Educational)
   - Primary target personas (ask for 2-3 key personas)
   - Brand voice characteristics (tone, style preferences)

2. **Create SQLite Database**:
   - Run the migration script at `${CLAUDE_PLUGIN_ROOT}/migrations/001_initial_schema.sql`
   - Initialize all required tables (content, backlog, schedules, alerts, metrics)
   - Record initialization timestamp

3. **Scaffold Configuration Files**:
   - Copy templates from `${CLAUDE_PLUGIN_ROOT}/config-templates/` to the working directory
   - Populate the following YAML files with wizard answers:
     - `config.yaml` - site domain, CMS type, strategy
     - `personas.yaml` - target personas and characteristics
     - `brand_voice.yaml` - brand voice profile
     - `qa.yaml` - quality gates (default: solo profile)
     - `scoring.yaml` - backlog prioritization weights

4. **Detect MCP Capabilities**:
   - Check for available MCP servers (Search Console, Analytics, CMS connectors, etc.)
   - Record available capabilities in `config.yaml` under `mcp_capabilities`

5. **Create Initial Scaffolding**:
   - Create empty backlog
   - Create default workflows
   - Log successful initialization

### 3. Brownfield Path (Existing Site)

For existing installations, offer to run discovery:

1. Detect existing configuration files
2. Validate the existing database
3. Ask if user wants to run `/discover` to analyze existing content
4. If yes, trigger the discover command workflow
5. Present discovery findings for review and approval

### 4. Argument Handling

**--reconfigure**: Re-run the initialization wizard, overwriting configuration files (preserves database and content history)

**--repair**: Validate all configuration files against schemas and fix invalid entries. Validate database integrity and repair orphaned records where possible.

**--apply-overrides**: If a `config_overrides.yaml` exists, consolidate its values into the main YAML configuration files, then remove the overrides file.

## Success Criteria

- SQLite database created with all schema tables
- All YAML configuration files populated and valid
- MCP capabilities detected and recorded
- Initial backlog created (empty for greenfield, populated for brownfield)
- No validation errors in final configuration
- Plugin ready to accept commands (/status, /discover, /publish, etc.)

## Error Handling

- If database creation fails, provide diagnostic output and recovery options
- If configuration validation fails, identify the specific invalid entries
- If MCP detection fails, continue with degraded capabilities (record in alerts)
- If user cancels wizard mid-way, preserve any partial state and offer to resume

## Output

Print initialization summary showing:
- Installation type (greenfield/brownfield)
- Site domain and CMS type
- Personas and brand voice configured
- MCP capabilities detected
- Next recommended action (run /discover for brownfield, /publish for greenfield)
