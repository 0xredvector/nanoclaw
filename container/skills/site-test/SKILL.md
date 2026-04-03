---
name: site-test
description: Run validation tests on the content autopilot system
allowed-tools: Read, Bash(*), Grep, Glob
---

## Paths

- **Site root:** `/workspace/extra/site`
- **Plugin root:** `/workspace/extra/plugin`
- **Scripts:** `/workspace/extra/plugin/scripts`
- **Prompts:** `/workspace/extra/plugin/prompts`
- **Config templates:** `/workspace/extra/plugin/config-templates`
- **Database:** `/workspace/group/content-autopilot.db`
- **Reports:** `/workspace/group/reports`

# Test Content Autopilot System

Run system validation tests to verify configuration, connectivity, data integrity, and operational readiness.

## Execution Flow

### 1. Test Selection Logic

Parse the argument to determine which test suites to run:
- **--config**: Configuration file validation only
- **--connections**: MCP server health checks only
- **--playbooks**: Playbook rule validation only
- **--prompts**: Prompt template validation only
- **--integrity**: Database integrity checks only
- **--full**: Run all test suites
- If no argument provided: default to --full

### 2. Configuration Validation (--config)

Validate all YAML configuration files against their schemas:

1. **Load Schema Files**:
   - Read schema definitions from `${CLAUDE_PLUGIN_ROOT}/config-templates/schemas/`
   - Load schemas for: config.yaml, personas.yaml, brand_voice.yaml, qa.yaml, scoring.yaml, workflows.yaml

2. **Validate Each Configuration File**:
   - For each YAML file in the working directory:
     - Parse YAML syntax (check for valid YAML)
     - Validate against corresponding schema
     - Check for required fields
     - Validate field types and constraints
     - Verify referential integrity (e.g., personas referenced in config exist in personas.yaml)
   - Report pass/fail with specific errors for each validation failure

3. **Cross-File Validation**:
   - Verify all referenced personas exist in personas.yaml
   - Verify all referenced workflows exist in workflows.yaml
   - Check that all CMS types are supported
   - Verify scoring weights sum to expected values

### 3. MCP Connections Health Check (--connections)

Test all configured MCP server connections:

1. **Read MCP Configuration**:
   - Load `${CLAUDE_PLUGIN_ROOT}/.mcp.json`
   - Read configured MCP servers from config.yaml

2. **For Each MCP Server**:
   - Attempt connection with timeout (10 seconds)
   - Execute a test method to verify functionality:
     - **Search Console**: Fetch last 7 days of data
     - **Analytics**: Fetch last 7 days of sessions
     - **CMS Connector**: List recent posts or pages
     - **Generic MCP**: Execute a discovery/info method
   - Record response time (latency)
   - Record success/failure status
   - Log any error messages

3. **Generate Connection Report**:
   - List each MCP server with status (✓ Connected, ✗ Failed, - Disabled)
   - Show latency for successful connections
   - Show error details for failed connections
   - Overall connectivity score

### 4. Playbook Rules Validation (--playbooks)

Test playbook rules against recent data:

1. **Load Playbook Files**:
   - Read all `.yaml` files from `${CLAUDE_PLUGIN_ROOT}/playbooks/` directory
   - Parse rule definitions and conditions

2. **For Each Playbook**:
   - Extract rules and conditions
   - Load recent content data from SQLite database (last 100 pieces)
   - For each rule:
     - Test condition evaluation logic
     - Verify rule actions are defined
     - Check for unreachable rules or logical conflicts
     - Execute rule against sample data to verify it produces expected results
   - Record pass/fail for each rule

3. **Generate Playbook Report**:
   - List all playbooks and their validation status
   - Show any syntax errors or logical issues
   - Show test results (conditions evaluated, actions triggered)

### 5. Prompt Templates Validation (--prompts)

Validate all prompt template files:

1. **Locate Prompt Templates**:
   - Recursively scan `${CLAUDE_PLUGIN_ROOT}/prompts/` directory
   - Collect all `.md` files

2. **For Each Prompt Template**:
   - Parse YAML frontmatter
   - Check for required fields (description, model, temperature, etc.)
   - Verify template variables are syntactically valid (e.g., {{ variable }})
   - Test template rendering with sample data:
     - Create minimal sample persona, brand voice, content data
     - Attempt to render template with these variables
     - Verify output is non-empty and structurally valid
   - Check for missing or undefined template variables
   - Verify file encoding is UTF-8

3. **Generate Prompt Report**:
   - List all prompts with validation status
   - Show any template syntax errors
   - Show any missing variable definitions
   - Show sample rendered output for a few key prompts

### 6. Database Integrity Checks (--integrity)

Validate SQLite database schema and data:

1. **Schema Validation**:
   - Verify all required tables exist:
     - system_config, content, backlog, schedules, alerts, metrics, workflow_runs, mcp_events
   - Verify all required columns exist in each table
   - Verify column types match schema expectations
   - Check for unexpected tables or columns

2. **Foreign Key Validation**:
   - Enable PRAGMA foreign_keys
   - For each foreign key relationship, verify:
     - All referenced parent records exist
     - No orphaned child records exist
   - Report any foreign key violations

3. **Data Invariant Checks**:
   - Verify content status values are valid (draft, review, published, archived)
   - Verify backlog priority scores are in valid range (0-100)
   - Verify metric values are positive numbers
   - Check for duplicate content pieces (by URL)
   - Verify timestamps are chronologically valid

4. **Orphan Detection**:
   - Identify content without corresponding backlog entries
   - Identify workflow runs referencing non-existent content
   - Identify alerts with invalid severity levels
   - Report findings with remediation options

5. **Data Statistics**:
   - Count of records in each major table
   - Database file size
   - Index statistics
   - Query execution plan samples (EXPLAIN for key queries)

### 7. Test Reporting

For each test suite run, provide:

1. **Summary Section**:
   - Test suite name
   - Number of tests run
   - Number passed/failed/skipped
   - Overall result (✓ All Pass, ⚠ Warnings, ✗ Failures)

2. **Detailed Results**:
   - For each failed test: test name, error message, remediation suggestion
   - For each warning: description and context
   - Show stack traces for critical failures

3. **Remediation Guidance**:
   - Suggest next steps to fix failures
   - Recommend `/init --repair` if configuration issues found
   - Recommend manual database repair if integrity issues found

4. **Overall System Health**:
   - Indicate if system is ready for operations (all tests pass)
   - Identify any degraded capabilities (some connections failed, etc.)
   - Recommend `/init --reconfigure` if major issues found

## Success Criteria

- All configuration files are valid YAML with correct schemas
- All MCP connections are established (or marked as disabled)
- All playbook rules evaluate correctly against test data
- All prompt templates render without errors
- Database has no foreign key violations or orphaned records
- No critical errors are reported

## Error Handling

- If a test suite is unavailable, skip it and note as skipped
- If test fixtures cannot be created, report and continue with other tests
- If database is corrupted, provide detailed diagnostic and recommend `/init --repair`
- Do not modify any data during testing (read-only operations only)

## Output

Print a comprehensive test report with per-suite summaries and an overall system health indicator. Exit with code 0 for all-pass, 1 for warnings/failures.
