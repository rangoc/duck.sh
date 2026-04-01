# Duck Hour Logging — Design Spec (Part 2)

## Context

Part 1 extracts daily standup accomplishments from Slack into `data/standups.json`. Part 2 takes that data and logs hours on duck.dlabs.si via Chrome browser automation. The user must already be logged into duck.dlabs.si in Chrome.

## Architecture

A Claude Code skill (`/duck-log`) that uses Chrome MCP browser tools to interact with duck.dlabs.si. Reads standup data and config, then fills in the activity form for each day.

## Inputs

- `data/standups.json` — daily accomplished text
- `config.json` — ticket IDs, hour allocations, education Fridays

## Per-Day Entries

### Standard day (Mon–Fri)

| Ticket | Description | Hours |
|--------|-------------|-------|
| Development (from config) | accomplished text from standups.json | 6.5h |
| Daily/PM (from config) | "Dailies" | 1.0h |
| **Total** | | **7.5h** |

### Education Friday

| Ticket | Description | Hours |
|--------|-------------|-------|
| Development (from config) | accomplished text from standups.json | 4.0h |
| Daily/PM (from config) | "Dailies" | 1.0h |
| Education (from config) | "AI Initiatives" | 2.5h |
| **Total** | | **7.5h** |

## Skip Logic

- Days with `provisional: true` in standups.json → skip (not yet confirmed)
- Days already showing >0h on duck.dlabs.si calendar → skip (already logged)
- Vacation/holiday days → not in standups.json, naturally skipped

## Browser Automation Flow

### Prerequisites

- User must be logged into duck.dlabs.si in Chrome
- `data/standups.json` must exist (run `/duck-extract` first)
- `config.json` must exist

### Step-by-step per entry

1. Navigate to duck.dlabs.si (or confirm already there)
2. Read the calendar bar to identify which days have 0h (need logging)
3. For each day that needs logging and has standup data:
   a. Select the ticket (see Ticket Selection below)
   b. The ticket form loads — fill in the "Activity description..." textbox with the description
   c. Select the hours from the time dropdown (value in minutes: 6.5h = 390, 4h = 240, 1h = 60, 2.5h = 150)
   d. Change the date field from "activity happened today" to the target date
   e. Click the "Add" button
   f. Wait for the page to update
   g. Repeat for the next ticket entry on the same day (Daily/PM, and Education if applicable)
4. Move to the next day

### Ticket Selection

Do NOT assume tickets are in the Favourite tickets sidebar. Instead:

1. Check if the ticket exists in the Favourite tickets list on the page
2. If found: click it directly
3. If not found: use the "Find a ticket..." search box to search by ticket name/ID, then select it from results

This ensures the automation works for any user, regardless of their favourites setup.

### Form field mapping

- Ticket selection: search or click favourite (see above)
- Description: textbox with placeholder "Activity description..."
- Hours: combobox/dropdown with values in minutes (e.g., "06:30" = value "390")
- Date: textbox showing "activity happened today" — needs to be changed to target date format
- Submit: button "Add"

## Skill Structure

```
.claude/skills/duck-log/
├── SKILL.md          # Skill metadata and trigger
└── instructions.md   # Step-by-step browser automation workflow
```

## Verification

1. Run `/duck-log` with a single test day
2. Verify the correct ticket, description, hours, and date were submitted
3. Check duck.dlabs.si shows the new entry
4. User cleans up the test entry manually
5. After verification: run on remaining days
