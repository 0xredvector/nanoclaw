---
name: media
description: Manage media assets, alt text, and image optimization
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

# Media Management

Manage media assets, generate alt text, and optimize images for content pieces.

## Execution Flow

### 1. Argument Handling

- **content_id**: Show/manage media for a specific content piece
- **--manifest**: Generate media manifest for next unpublished content
- **--alt-text**: Audit and generate missing alt text across all content
- **--optimize**: Show optimization recommendations for existing media

### 2. Content Media (content_id)

1. Load content piece from SQLite
2. Load or generate media manifest via `MediaProducer.generate_manifest()`
3. Display manifest:
   ```
   Media Manifest — "Getting Started with FastAPI"
   ════════════════════════════════════════════════

   Slot  | Type       | Status    | Alt Text
   ──────┼────────────┼───────────┼──────────────
   hero  | hero_image | missing   | ✗ needed
   fig-1 | diagram    | assigned  | ✓ set
   fig-2 | screenshot | missing   | ✗ needed
   ```
4. For missing alt text: generate via alt-text-writer prompt
5. Ask: "Generate alt text for N images?"

### 3. Manifest Mode (--manifest)

1. Find next content piece with status = draft or review
2. Generate full media manifest based on content type
3. List required assets with specifications (dimensions, format, purpose)
4. Show optimization specs (WebP, lazy loading, responsive sizes)
5. Save manifest to DB

### 4. Alt Text Audit (--alt-text)

1. Scan all published content for media items
2. Identify items missing alt text
3. Display summary:
   - Total media items
   - Items with alt text
   - Items missing alt text
   - Items with generic alt text (e.g., "image1.png")
4. Offer to batch-generate alt text using alt-text-writer prompt

### 5. Optimization Mode (--optimize)

1. Scan all media items for optimization opportunities:
   - Non-WebP images (suggest conversion)
   - Missing lazy loading attributes
   - Missing responsive sizes
   - Oversized images (width > 1200px)
   - Missing compression
2. Generate optimization report with estimated page speed improvement

## Error Handling

- If no content found: suggest running /inventory first
- If media module unavailable: show basic file listing from content
- If alt text generation fails: save partial results, flag for manual review

## Output

Display media status and recommendations. Save manifests and alt text to database.
