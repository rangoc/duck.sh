# Duck Fill — Step-by-Step Workflow

Follow these steps exactly when `/duck-fill` is invoked.

This skill is the Slack-free twin of `/duck-extract`. It writes the same
`data/standups.json` structure, but the `accomplished` text is always the
literal string `"Work"`. No Slack MCP tools are used or required.

## Step 0: Load Configuration

Read `config.json` from the project root.

- If it **does not exist**, run the **First-Run Setup** below, then continue.
- If it exists, parse it and proceed to Step 1.

### First-Run Setup

Only the ticket/hour parts of `config.json` matter here — the Slack fields are
unused by this skill, but `/duck-log` and `/duck-extract` share the same file,
so write them as empty placeholders rather than omitting them.

1. **Check dependencies.** This skill needs no MCP servers. `/duck-log` (the
   next step in the workflow) needs Chrome MCP, so check whether
   `mcp__claude-in-chrome__tabs_context_mcp` **exists** in your tool list —
   check for tool *existence*, not call *success*. Print:

   ```
   ┌─────────────────────────────────────────────┐
   │  duck.sh — Initial Setup (fill mode)        │
   │                                              │
   │  Checking dependencies...                    │
   │                                              │
   │  [OK]      Chrome MCP   — available          │
   │  [SKIP]    Slack MCP    — not needed         │
   │                                              │
   │  Let's configure your tickets.               │
   └─────────────────────────────────────────────┘
   ```

   If Chrome MCP is missing, **continue** with setup but tell the user
   `/duck-log` won't work until they install it (extension: claude.ai/chrome,
   log in with the same account as Claude Code, restart Chrome).

2. Ask for the user's name (used only as a label in the output file).
   **Wait for the answer.**

**IMPORTANT: Ask the following questions ONE AT A TIME. Ask, WAIT for the
answer, then move on. Never batch questions.**

3. Ask about ticket mappings, the same way `/duck-extract` does:

   > "For `/duck-log` to log your hours on duck.dlabs.si, I need to know which
   > tickets you use and how many hours to allocate to each.
   >
   > A typical setup looks like:
   > - A **development** ticket for your main billable work (e.g., 6.5h/day) — description pulled from the standup data
   > - A **daily/standup** ticket for meetings and standups (e.g., 1h/day) — description: "Dailies"
   > - An optional **education** ticket for learning days (e.g., 2.5h on certain Fridays) — description: "AI Initiatives"
   >
   > This is just a suggestion — as many or as few tickets as you need.
   >
   > To find your ticket IDs, log into https://duck.dlabs.si/ and check your
   > Favourite tickets or search for them.
   >
   > Tell me about your ticket setup — which tickets, what IDs, how many hours each per day?"

   - **Wait for the answer.** Parse whatever format they give and adapt the
     config structure to match. Ask a follow-up if anything is unclear.

4. Ask about special Fridays:

   > "Are there any Fridays this month where your hours split differently?
   > (e.g., education days, team events) Day numbers like '6, 13, 20' or 'none'."

   - **Wait for the answer.** Accept day numbers, ordinals, or full dates;
     convert to `YYYY-MM-DD` in the current month.

5. Write `config.json`:

```json
{
  "slack_channel": "",
  "standup_bot_name": "",
  "user_name": "<from step 2>",
  "user_id": "",
  "tickets": {
    "<role>": { "id": "<id>", "name": "<name>", "default_hours": <hours>, "description": "<fixed text or 'standup'>" }
  },
  "special_fridays": {
    "dates": ["YYYY-MM-DD", ...],
    "adjustments": { "<ticket_role>": <adjusted_hours> }
  }
}
```

A ticket's `description` of `"standup"` means `/duck-log` uses the
`accomplished` text from `standups.json` — which, in fill mode, is `"Work"`.

## Step 1: Refresh Month-Specific Settings

Same rules as `/duck-extract`. Two things are month-specific:
- **Vacation days** — stored in `data/standups.json` (`vacation_days`, `vacation_confirmed`)
- **Special Fridays** — stored in `config.json` under `special_fridays.dates`

### 1a. Detect month change

The month has rolled over if either:
- `data/standups.json` doesn't exist, OR
- its `month` field doesn't match the current `YYYY-MM`.

If neither is true AND `vacation_confirmed` is `true`, skip 1b — the existing
vacation list still applies. Likewise skip 1c if `special_fridays.dates` are
already in the current month.

### 1b. Vacation days

If the month rolled over OR `vacation_confirmed` is `false`/missing, ask:

> "Any vacation or holiday days this month to skip? Day numbers like '9, 15, 23', or 'none'."

**Wait for the answer.** Accept day numbers, ordinals, or full dates; convert
to `YYYY-MM-DD`. Store in `vacation_days` and set `vacation_confirmed: true`.

### 1c. Special Fridays

If the dates in `config.json`'s `special_fridays.dates` are from a stale month,
ask:

> "Are there any Fridays this month where your hours split differently? Day numbers like '6, 13, 20' or 'none'."

**Wait for the answer.** Write the updated `special_fridays.dates` back to
`config.json` — leave `special_fridays.adjustments` and every other field
untouched.

## Step 2: Determine the Date Range

Get today's date with `date +%Y-%m-%d` (do not assume it).

- `month` = current `YYYY-MM`
- Start = first day of the current month
- End = **today**, inclusive

Do not generate days in the future — the user hasn't worked them yet. If the
user explicitly asked to fill the whole month (e.g., "fill all of July"), use
the last day of the month as the end instead, and say so in the report.

If the user named a different month in their invocation (e.g.
`/duck-fill June`), use that month with the full month as the range, unless it
is the current month.

## Step 3: Enumerate Working Days

For each date from Start to End inclusive:

- **Skip Saturdays and Sundays.**
- **Skip dates in `vacation_days`.**

Everything left is a working day. Determine weekday reliably — use
`date -j -f %Y-%m-%d <date> +%u` (1=Mon … 7=Sun) on macOS rather than counting
by hand.

Generate no `warnings` — there is no standup source to be missing. The
`warnings` array is present but empty, for structural compatibility.

## Step 4: Merge With Existing Data

If `data/standups.json` exists and its `month` matches the target month:

- Keep every existing day record **as-is** when its `accomplished` is anything
  other than `"Work"` — real extracted text from `/duck-extract` must never be
  clobbered by a generic fill.
- For an existing record whose `accomplished` is `"Work"`, keep it but force
  `provisional: false`.
- Append a new record for every working day not already present.

If the month doesn't match (or the file is absent), start fresh.

Each generated record:

```json
{
  "date": "<YYYY-MM-DD>",
  "accomplished": "Work",
  "provisional": false
}
```

`provisional` is always `false` — there is nothing to confirm later, and
`/duck-log` skips provisional days.

## Step 5: Build Output

```json
{
  "extracted_at": "<current ISO 8601 timestamp, e.g. 2026-07-31T10:22:04Z>",
  "user": "<user_name from config>",
  "user_id": "<user_id from config, may be empty>",
  "channel": "<slack_channel from config, may be empty>",
  "month": "<YYYY-MM>",
  "source": "fill",
  "vacation_days": ["<date1>", "<date2>"],
  "vacation_confirmed": true,
  "days": [ ... ],
  "warnings": []
}
```

`source: "fill"` marks this file as generated rather than extracted. Every
other field matches `/duck-extract`'s schema exactly, so `/duck-log` reads it
unchanged. Sort `days` by date ascending.

Get the timestamp with `date -u +%Y-%m-%dT%H:%M:%SZ`.

## Step 6: Write Output

Create `data/` if it doesn't exist. Write the JSON to `data/standups.json`
with 2-space indentation.

## Step 7: Report

Print a summary:

```
Filled 23 working days for July 2026 (2026-07-01 → 2026-07-31).
- accomplished: "Work" for all generated days
- 2 vacation days skipped
- 4 existing extracted days preserved
Output: data/standups.json

Next: run /duck-log to log these hours.
```

Report only what actually happened — if nothing new was added, say so.
