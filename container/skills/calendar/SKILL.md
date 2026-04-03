---
name: calendar
description: Manage the editorial content calendar
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

# Editorial Calendar

View and manage the content publishing calendar.

## Execution Flow

### 1. Argument Handling

- **--view** (default): Show calendar for current and next 3 weeks
- **--schedule**: Interactively schedule content to open slots
- **--auto**: Auto-schedule from backlog based on priority and cluster rotation
- **--forecast**: Show capacity forecast and backlog runway
- **--blackout DATE**: Add a blackout date (no publishing)

### 2. View Mode (--view)

1. Load calendar slots via `CalendarManager.get_upcoming(days=28)`
2. Render ASCII calendar:
   ```
   Content Calendar — February/March 2026
   ══════════════════════════════════════════════════
   Mon    Tue    Wed    Thu    Fri    Sat    Sun
   ──────────────────────────────────────────────────
   23     24     25     26     27     28     1
          ✅            📋
          pub'd         sched
   2      3      4      5      6      7      8
   📝            ⬜           📋
   draft         open         sched
   ```
3. Show legend and slot details below calendar
4. Show pipeline summary: N published, N scheduled, N drafts, N open

### 3. Schedule Mode (--schedule)

1. Show open slots for next 4 weeks
2. Show top backlog items ready to schedule
3. Ask user to match content to slots
4. Validate:
   - No cluster conflicts (min gap between same cluster)
   - Content type variety
5. Save assignments

### 4. Auto-Schedule (--auto)

1. Load all open slots and prioritized backlog
2. Run `CalendarManager.auto_schedule()`:
   - Fill slots by priority score
   - Enforce cluster rotation (min_gap_between_same_cluster)
   - Mix content types
3. Display proposed schedule
4. Ask: "Accept this schedule? (You can adjust individual slots later)"

### 5. Forecast (--forecast)

1. Calculate via `CalendarManager.capacity_forecast()`:
   - Open slots in next 4/8/12 weeks
   - Backlog items ready for production
   - Current production velocity (articles/week)
   - Estimated backlog runway (weeks of content)
   - Cluster coverage gaps in upcoming schedule
2. Display:
   ```
   Capacity Forecast
   ════════════════════════════
   Open slots (4 weeks):  8
   Backlog ready:         23 items
   Production velocity:   2.1 articles/week
   Backlog runway:        ~11 weeks

   Cluster gaps: "DevOps" not scheduled in next 3 weeks
   ```

### 6. Blackout (--blackout DATE)

1. Parse date
2. Check if slot exists for that date
3. If content is scheduled: ask to reschedule
4. Add blackout via `CalendarManager.add_blackout()`
5. Confirm: "Blackout added for DATE. No content will be scheduled."

## Error Handling

- If no calendar configured: prompt to run configure first
- If backlog empty: show calendar with open slots, suggest /research
- If all slots filled: show next available opening

## Output

Display calendar view. Save schedule changes to database.
