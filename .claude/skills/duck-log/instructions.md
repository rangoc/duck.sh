# Duck Log — Step-by-Step Browser Automation Workflow

Follow these steps exactly when `/duck-log` is invoked.

## Step 0: Load Data

1. **Check dependencies.** Check that Chrome MCP tools **exist** in your available tool list (do NOT rely on whether a call succeeds — a tool can exist but fail transiently if Chrome isn't open). Also check that `config.json` and `data/standups.json` exist as files.

   Print a status report:
   ```
   ┌─────────────────────────────────────────────┐
   │  duck.sh — Log Hours                        │
   │                                              │
   │  Checking dependencies...                    │
   │                                              │
   │  [OK] Chrome MCP    — available              │
   │  [OK] config.json   — found                  │
   │  [OK] standups.json — found                  │
   │                                              │
   │  Ready to log hours on duck.dlabs.si         │
   └─────────────────────────────────────────────┘
   ```

   If Chrome MCP tool does **not exist** (MCP server not configured):
   ```
   ┌──────────────────────────────────────────────────────────┐
   │  duck.sh — Log Hours                                     │
   │                                                           │
   │  Checking dependencies...                                 │
   │                                                           │
   │  [MISSING] Chrome MCP (Claude in Chrome)                  │
   │            Required for browser automation on duck.dlabs.si│
   │                                                           │
   │            How to set up:                                  │
   │            1. Install extension: claude.ai/chrome          │
   │            2. Log into claude.ai in Chrome with the same   │
   │               account as Claude Code                       │
   │            3. Restart Chrome                               │
   │            4. Re-run /duck-log                             │
   └──────────────────────────────────────────────────────────┘
   ```

   Do not proceed if Chrome MCP is missing. If `config.json` or `data/standups.json` is missing, show `[MISSING]` for it and tell the user to run `/duck-extract` first.

2. Read `config.json` from the project root.
3. Read `data/standups.json`.
3. From config, extract ticket info from `tickets` object. Each ticket has an `id`, `name`, `default_hours`, and `description` field. Also read `special_fridays` for dates with adjusted hour splits. The config is flexible — there may be any number of tickets with any role names.

## Step 1: Determine Days to Log

From `data/standups.json`, collect all days where:
- `provisional` is `false`

These are the candidate days. Days with `provisional: true` are skipped.

## Step 2: Open duck.dlabs.si

1. Call `mcp__claude-in-chrome__tabs_context_mcp` to get available tabs.
2. Check if duck.dlabs.si is already open. If yes, use that tab. If not, create a new tab and navigate to `https://duck.dlabs.si`.
3. Call `mcp__claude-in-chrome__read_page` to read the calendar bar.

## Step 3: Identify Days Already Logged

Read the calendar bar on the duck.dlabs.si home page. Each day shows its total hours (e.g., "2026-03-02: 7.5h" or "2026-03-07: 0h").

From the candidate days (Step 1), **remove** any day that already shows >0h on duck. These are already logged — skip them.

The remaining days are the ones that need logging.

If no days need logging, report "All days are already logged" and stop.

## Step 4: Log Each Day

For each day that needs logging, add one entry per ticket defined in `config.tickets`.

### Entry types per day

**Standard day:** Log each ticket with its `default_hours`. For the description:
- If the ticket's `description` field is `"standup"` → use the `accomplished` text from standups.json for that day
- Otherwise → use the `description` string as-is (e.g., "Dailies")

**Special Friday** (date is in `config.special_fridays.dates`): Use the adjusted hours from `config.special_fridays.adjustments` for tickets that have overrides. Tickets not listed in adjustments keep their `default_hours`. Any additional tickets specified only in `special_fridays.adjustments` are added on these days.

### Adding a single entry

For each entry, follow this exact sequence:

#### 4a. Search and select the ticket

1. Click the "Find a ticket..." search box on the page.
2. Clear any existing text (select all + delete).
3. Type the Jira ticket key from the ticket's `name` field in config (e.g., the part like "BOFTWO-3587" or "INTERNAL-9").
4. Wait briefly for search results to appear.
5. Click on the matching result in the dropdown. The page will navigate to the ticket view with the activity form.

#### 4b. Fill in the description

1. Click the "Activity description..." textarea.
2. Type the description text based on the ticket's `description` field in config:
   - If `"standup"` → use the `accomplished` string from standups.json for that day
   - Otherwise → use the `description` string as-is

#### 4c. Set the hours

Use `mcp__claude-in-chrome__form_input` on the time combobox (dropdown) with the value in **minutes**:

| Hours | Minutes value |
|-------|--------------|
| 1.0h  | 60           |
| 2.5h  | 150          |
| 4.0h  | 240          |
| 5.0h  | 300          |
| 5.25h | 315          |
| 6.0h  | 360          |
| 6.5h  | 390          |
| 7.0h  | 420          |
| 7.5h  | 450          |
| 8.0h  | 480          |

To convert hours to the dropdown value: multiply hours by 60. E.g., 6.5 * 60 = 390.

#### 4d. Set the date

The form has a date field that defaults to "activity happened today". If the target day is NOT today:

1. Click the date text field.
2. Clear it and type the date in the format shown on the page (check what format duck uses — typically `YYYY-MM-DD` or the locale format).
3. Confirm the date is set correctly.

If the target day IS today, leave it as-is.

#### 4e. Submit

1. Click the "Add" button.
2. Wait for the page to update (take a screenshot to confirm).
3. Verify the entry appears in the activity list.

#### 4f. Repeat

After adding one entry, the page navigates back to the home view. Repeat from step 4a for the next entry on the same day (Daily/PM, Education if applicable), then move to the next day.

## Step 5: Report

After all entries are added, print a summary:
- Number of days logged
- Number of entries added (days * entries per day)
- Any days that were skipped (already logged or provisional)
- Remind the user to verify on duck.dlabs.si

Example:
```
Logged 5 days on duck.dlabs.si:
- 10 entries added (5 days x 2 entries)
- 1 education Friday (3 entries)
- 17 days skipped (already logged)
- 0 provisional days skipped
Verify at: https://duck.dlabs.si
```

## Error Handling

- If the browser extension disconnects: inform the user and stop. They can re-run the skill to continue — already-logged days will be skipped.
- If a ticket search returns no results: warn the user and skip that entry.
- If the form doesn't load: take a screenshot, inform the user, and stop.
