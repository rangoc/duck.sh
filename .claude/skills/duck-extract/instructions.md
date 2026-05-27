# Duck Extract — Step-by-Step Workflow

Follow these steps exactly when `/duck-extract` is invoked.

## Step 0: Load Configuration

Read `config.json` from the project root.

- If it **does not exist**, run the **First-Run Setup** (see below), then continue.
- If it exists, parse it and proceed to Step 1.

### First-Run Setup

When no `config.json` exists, walk the user through setup:

1. **Check MCP dependencies.** Before anything else, check that both required MCP integrations are available.

   **How to check:** A tool is available if it exists in your tool list — do NOT rely on whether a call succeeds or fails. A tool can exist (MCP server is connected) but a call might fail for transient reasons (e.g., Chrome isn't open). Check for tool **existence**, not call **success**.

   - **Slack MCP**: Check if `mcp__claude_ai_Slack__slack_read_user_profile` exists as an available tool.
   - **Chrome MCP**: Check if `mcp__claude-in-chrome__tabs_context_mcp` exists as an available tool.

   Then print a status report:

   ```
   ┌─────────────────────────────────────────────┐
   │  duck.sh — Initial Setup                    │
   │                                              │
   │  Checking dependencies...                    │
   │                                              │
   │  [OK]      Slack MCP    — available           │
   │  [OK]      Chrome MCP   — available           │
   │                                              │
   │  All good! Let's configure your workspace.   │
   └─────────────────────────────────────────────┘
   ```

   If a tool does **not exist** (MCP server not configured at all):
   ```
   ┌──────────────────────────────────────────────────────────┐
   │  duck.sh — Initial Setup                                 │
   │                                                           │
   │  Checking dependencies...                                 │
   │                                                           │
   │  [MISSING] Slack MCP                                      │
   │            Required for /duck-extract (standup extraction) │
   │                                                           │
   │            How to set up:                                  │
   │            1. Type /mcp in Claude Code                     │
   │            2. Add the Slack MCP server                     │
   │            3. Follow: docs.slack.dev/ai/slack-mcp-server/       │
   │               connect-to-claude/#connect-to-claude-code-via-plugin │
   │                                                           │
   │  [MISSING] Chrome MCP (Claude in Chrome)                  │
   │            Required for /duck-log (hour logging)           │
   │                                                           │
   │            How to set up:                                  │
   │            1. Install extension: claude.ai/chrome          │
   │            2. Log into claude.ai with same account         │
   │               as Claude Code                               │
   │            3. Restart Chrome                               │
   │                                                           │
   │  Fix the above, then re-run /duck-extract                 │
   └──────────────────────────────────────────────────────────┘
   ```

   If Slack MCP is missing, **stop** — it's required for this skill. If only Chrome MCP is missing, **continue** with setup but note that `/duck-log` won't work until it's installed.

2. From the successful Slack profile call, extract the current user's name and Slack ID.

**IMPORTANT: The following questions must be asked ONE AT A TIME. Ask a question, WAIT for the user's answer, then proceed to the next question. Never batch multiple questions or move ahead before receiving a response.**

3. Ask: "What Slack channel contains your team standups?" (e.g., `bof-dev-pipe-paid-products`)
   - **Wait for answer.** Then call `mcp__claude_ai_Slack__slack_search_channels` with their answer.
   - Show the match and ask: "Is this the right channel?"
   - **Wait for confirmation** before continuing.

4. Ask: "What is the standup bot's display name?" (default: `Team Standup`)
   - **Wait for answer.**

5. Ask about ticket mappings for hour logging. Present this as a conversation, not a rigid form:

   > "For `/duck-log` to log your hours on duck.dlabs.si, I need to know which tickets you use and how many hours to allocate to each.
   >
   > A typical setup looks like:
   > - A **development** ticket for your main billable work (e.g., 6.5h/day) — description pulled from your standup
   > - A **daily/standup** ticket for meetings and standups (e.g., 1h/day) — description: "Dailies"
   > - An optional **education** ticket for learning days (e.g., 2.5h on certain Fridays) — description: "AI Initiatives"
   >
   > But this is just a suggestion — you can have as many or as few tickets as you need, with whatever hour splits work for you.
   >
   > To find your ticket IDs, log into https://duck.dlabs.si/ and check your Favourite tickets or search for them.
   >
   > Tell me about your ticket setup — which tickets do you use, what are their IDs, and how many hours go to each per day?"

   - **Wait for answer.** The user may describe their setup in any format — structured or free-form. Parse whatever they provide and adapt the config structure to match. Some users may have 2 tickets, others may have 5. Some may have special rules for certain days. Record all of it.
   - If anything is unclear, ask a follow-up. Don't assume.

6. Ask about education or special Fridays:

   > "Are there any Fridays this month where your hours split differently? (e.g., education days, team events, etc.) If so, which ones? You can just list the day numbers like '6, 13, 20' or 'none'."

   - **Wait for answer.** Accept flexible date input — day numbers (6, 13, 20), ordinals (6th, 13th), or full dates. Convert to `YYYY-MM-DD` format using the current month.

7. Write `config.json` with the collected info. The structure should accommodate whatever tickets the user described:

```json
{
  "slack_channel": "<channel-name>",
  "standup_bot_name": "Team Standup",
  "user_name": "<from Slack profile>",
  "user_id": "<from Slack profile>",
  "tickets": {
    "<role>": { "id": "<id>", "name": "<name>", "default_hours": <hours>, "description": "<fixed text or 'standup'>" },
    ...
  },
  "special_fridays": {
    "dates": ["YYYY-MM-DD", ...],
    "adjustments": {
      "<ticket_role>": <adjusted_hours>,
      ...
    }
  }
}
```

The `tickets` object is flexible — keys are whatever roles the user described (development, daily, education, etc.). Each ticket's `description` field indicates what text to use when logging: `"standup"` means use the accomplished text from standups.json, any other string is used as-is (e.g., `"Dailies"`).

## Step 1: Refresh Month-Specific Settings

Two pieces of config are month-specific and must be refreshed at the start of each new month:
- **Vacation days** — stored in `data/standups.json` (`vacation_days` + `vacation_confirmed`).
- **Special Fridays** — stored in `config.json` under `special_fridays.dates`.

The rest of `config.json` (channel, user, tickets, hour adjustments) is durable across months.

### 1a. Detect month change

The month has rolled over if either:
- `data/standups.json` doesn't exist, OR
- its `month` field doesn't match the current `YYYY-MM`.

If neither is true AND `vacation_confirmed: true`, skip both prompts below — existing values are still valid for this month.

### 1b. Vacation days

If the month rolled over OR `vacation_confirmed` is `false`/missing, ask:

> "Any vacation or holiday days this month to skip? You can list day numbers like '9, 15, 23' or just say 'none'."

**Wait for answer** before proceeding. Accept flexible input — day numbers (9, 15), ordinals (9th, 15th), or full dates. Convert to `YYYY-MM-DD` format using the current month. Store in `vacation_days` and set `vacation_confirmed: true` in the output JSON.

### 1c. Special Fridays

If the month rolled over (i.e., the dates in `config.json`'s `special_fridays.dates` are from a different month, or the array is from a stale month), ask:

> "Are there any Fridays this month where your hours split differently? (e.g., education days, team events) Day numbers like '6, 13, 20' or 'none'."

**Wait for answer.** Accept day numbers, ordinals, or full dates and convert to `YYYY-MM-DD` using the current month. If the user says "none", set `dates` to `[]`. Write the updated `special_fridays.dates` back to `config.json` — leave `special_fridays.adjustments` and every other field in `config.json` untouched.

## Step 2: Find the Channel

Call `mcp__claude_ai_Slack__slack_search_channels` with the configured `slack_channel` name.

Extract the `channel_id` from the result. If not found, warn the user and stop.

## Step 3: Fetch Standup Bot Messages

Determine `oldest_date` (incremental optimization — avoids re-fetching the whole month on every run):

- If `data/standups.json` exists AND its `month` matches the current month AND it has a non-empty `days` array:
  - Let `provisional_dates` = dates of all days where `provisional: true`.
  - If `provisional_dates` is non-empty: `oldest_date` = earliest provisional date. (We need to re-resolve these once their next-day reply arrives.)
  - Otherwise: `oldest_date` = the day after the latest date in `days[]`. (No provisional days to confirm — only fetch new days.)
  - If `oldest_date` is in the future (i.e., everything is already extracted and confirmed): skip Steps 3–6 and go straight to Step 7 with the existing data unchanged. Report "nothing new to extract" in Step 9.
- Otherwise (first run, or `month` mismatch): `oldest_date` = first day of the current month.

Calculate timestamps:
- `oldest`: `oldest_date` at 00:00:00 in the user's local timezone, as Unix seconds. (Local-midnight is safer than UTC-midnight — it guarantees we catch the bot message posted at ~08:00 local time.)
- `latest`: current timestamp.

Call `mcp__claude_ai_Slack__slack_read_channel` with:
- `channel_id`: from Step 2
- `oldest`: Unix timestamp from above
- `latest`: now Unix timestamp
- `limit`: 100

From the results, filter for messages posted by the standup bot. Identify these by checking the message sender's name or bot label matching the configured `standup_bot_name` (e.g., "Team Standup"). These are the parent messages.

If there are more than 100 messages in the channel for the window, paginate using the `cursor` field until all messages are retrieved.

Collect each standup bot message's `ts` (timestamp) — these are needed to read the threads.

## Step 4: Read Each Thread

For each standup bot message `ts` from Step 3:

Call `mcp__claude_ai_Slack__slack_read_thread` with:
- `channel_id`: from Step 2
- `message_ts`: the bot message's `ts`
- `limit`: 100

From the thread replies, find replies where the sender's user ID matches the configured `user_id`. If the user has multiple replies in a thread, use the **last one** (most recent).

**Fallback**: If `mcp__claude_ai_Slack__slack_read_thread` returns a `thread_not_found` error, fall back to `mcp__claude_ai_Slack__slack_search_public_and_private` with query `from:<@USER_ID> in:#CHANNEL on:YYYY-MM-DD` to find the user's reply for that date.

If no reply from the user is found for a given day, note it as a missing day.

## Step 5: Parse Replies

For each user reply found in Step 4, parse the message text into three parts:

The expected format is a numbered list:
```
1. <yesterday items>
2. <today items>
3. <blockers>
```

Parsing rules:
- **Item 1** (text after "1." until "2."): yesterday — what was done the previous day
- **Item 2** (text after "2." until "3." or end): today — what is planned for today
- **Item 3** (text after "3." to end, if present): blockers

Each item may contain sub-items, bullet points, or free-form text. Preserve the content but strip the leading number and period.

**Vacation detection**: If the entire reply (case-insensitive) contains only words like "vacation", "holiday", "day off", "sick day", "pto", or similar short phrases indicating time off, flag this day as a vacation day.

**Unparseable replies**: If a reply does not match the expected numbered format (1./2./3.), attempt best-effort extraction. If no meaningful content can be parsed, skip the day and add a warning.

**Date from timestamp**: Convert the parent bot message's Slack `ts` (Unix timestamp, e.g., `1711929600.000000`) to a calendar date (`YYYY-MM-DD`) in the user's local timezone. The integer part before the dot is seconds since epoch.

## Step 6: Resolve Accomplished

Build a map of `date → parsed reply` from Step 5.

For each day D that has a reply:

1. Find the **next day in the map after D** (not necessarily D+1 — for Fridays, the next entry is typically Monday). Call this day N.
   - **N exists AND N is the next expected working day after D** (i.e., no missing working days in between): `accomplished` = N's "yesterday" content. Set `provisional: false`.
   - **N doesn't exist** (D is the last day in the map), OR **there's a gap** (missing working days between D and N, so N's "yesterday" describes a different day): `accomplished` = D's own "today" content (item 2). Set `provisional: true`.

   **Exception — last working day of the month**: Always set `provisional: false`, even if there's no D+1. Reason: next month's run overwrites standups.json, so we'll never get confirmation. Treat the day's own "today" as final.

   **Exception — last working day before vacation/holiday block**: Always set `provisional: false`. If every day between D and N is a vacation day (from `vacation_confirmed` list) or a public holiday, N's "yesterday" will describe the vacation ("Vacation", "Holiday", etc.) — not D's work. The user will never confirm D after the fact, so treat D's own "today" as final rather than blocking it as provisional. This applies when D is the last standup before a stretch of vacation/holiday days, regardless of how long the stretch is.

   Note: weekends are not gaps — Friday → Monday is consecutive for this purpose.

2. Clean up the accomplished text:
   - Strip leading/trailing whitespace
   - Remove bullet point markers if present
   - If multiple sub-items exist, join them with "; " (semicolons)
   - Result should be a clean, readable string like: "Send newsletter invite query / Retool app; Admin app: Invite to subscribe tool"

## Step 7: Build Output

Skip these days entirely (do not include in the `days` array):
- Days flagged as vacation/holiday (auto-detected in Step 5)
- Days listed in the user-declared `vacation_days`
- Days where the user had no reply (but add a warning)

For missing **weekdays** (Monday–Friday) that are NOT vacation: add to the `warnings` array: `"<date>: No standup reply found"`. Do NOT generate warnings for Saturdays or Sundays — those are expected to have no standups.

**Merge with existing data (incremental runs)**: If Step 3 used an `oldest_date` past the start of the month (i.e., this is an incremental run), merge as follows:
- Start from the existing `days[]` array.
- For each newly-extracted day record, replace the existing record for the same date if present, otherwise append.
- Recompute `warnings` from scratch over the full month — do not preserve stale warnings from the previous run.
- Re-sort `days[]` by date ascending after merging.

Build the output JSON:

```json
{
  "extracted_at": "<current ISO 8601 timestamp>",
  "user": "<user_name from config>",
  "user_id": "<user_id from config>",
  "channel": "<slack_channel from config>",
  "month": "<YYYY-MM>",
  "vacation_days": ["<date1>", "<date2>"],
  "vacation_confirmed": true,
  "days": [
    {
      "date": "<YYYY-MM-DD>",
      "accomplished": "<cleaned string>",
      "provisional": false
    }
  ],
  "warnings": [
    "<date>: No standup reply found"
  ]
}
```

Sort the `days` array by date ascending.

## Step 8: Write Output

Create the `data/` directory if it doesn't exist.

Write the JSON to `data/standups.json` with 2-space indentation. Re-running the skill overwrites this file — only the current month matters. Incremental runs preserve already-confirmed days from the previous run (see merge logic in Step 7); a new month's run replaces the file entirely.

## Step 9: Report

Print a summary to the user:
- Number of days extracted
- Number of provisional days (last day only, typically)
- Number of vacation days skipped
- Any warnings about missing days
- Path to the output file

Example:
```
Extracted 22 days for March 2026.
- 1 provisional (today — will be confirmed tomorrow)
- 2 vacation days skipped
- 1 warning: 2026-03-17 — no standup reply found
Output: data/standups.json
```
