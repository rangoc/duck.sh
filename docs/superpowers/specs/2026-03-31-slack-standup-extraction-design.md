# Slack Standup Extraction — Design Spec

## Context

d.labs teams post daily standups in Slack via a "Team Standup" bot workflow. The bot posts a parent message each morning, and team members reply in the thread with:

1. What they worked on yesterday
2. What they plan to work on today
3. Any blockers

This tool extracts a user's standup replies from Slack and produces a structured JSON file showing what was actually accomplished each day. The key insight: **tomorrow's "yesterday" is the source of truth for today's "today"** — people report what they actually did more accurately in retrospect.

This is Part 1 of **duck.sh**, a larger automation that will eventually log hours on duck.dlabs.si. Part 2 (browser automation for hour logging) is out of scope for this spec.

## Architecture

A **Claude Code skill** (`/duck-extract`) that uses Slack MCP tools. No external API keys or dependencies — leverages the existing Slack MCP connection available in Claude Code.

### Components

1. **Config loader** — reads `config.json`, triggers interactive setup on first run
2. **Slack fetcher** — finds the channel, locates standup bot messages, reads threads
3. **Parser** — extracts items 1/2/3 from user replies, detects vacation keywords
4. **Resolver** — cross-references adjacent days to determine actual accomplishments
5. **Writer** — outputs structured JSON to `./data/standups.json`

## Configuration

### `config.json` (project root)

```json
{
  "slack_channel": "bof-dev-pipe-paid-products",
  "standup_bot_name": "Team Standup",
  "tickets": {
    "development": {
      "id": "#97976",
      "name": "[BOFTWO-3587] - Memberships: Development",
      "default_hours": 6.5
    },
    "daily": {
      "id": "#97978",
      "name": "[BOFTWO-3589] - Daily / Project Management",
      "default_hours": 1.0
    },
    "education": {
      "id": "#33851",
      "name": "[INTERNAL-9] - Education",
      "hours": 2.5
    }
  },
  "education_fridays": ["2026-03-20", "2026-04-03"]
}
```

Ticket config is stored now for Part 2. Part 1 only uses `slack_channel` and `standup_bot_name`.

### First-run interactive setup

On first run (no `config.json` found), the skill:

1. Calls `slack_read_user_profile` to identify the current user (name + ID)
2. Asks which channel contains standups
3. Calls `slack_search_channels` to find and confirm the channel
4. Asks for ticket mappings (for Part 2, stored ahead of time)
5. Asks about education Fridays
6. Writes `config.json`

## Data Flow

### Manual trigger: `/duck-extract`

Every run processes the **full current month** (day 1 → today). The output JSON is fully overwritten each time.

```
1. Read config.json
2. slack_search_channels → find channel_id
3. slack_read_channel (oldest=month_start, latest=now) → all messages
4. Filter for messages from "Team Standup" bot → list of parent messages with timestamps
5. For each standup parent message:
   a. slack_read_thread(channel_id, message_ts) → thread replies
   b. Filter replies by current user's Slack ID
   c. Parse reply into: yesterday (item 1), today (item 2), blockers (item 3)
   d. Detect vacation/holiday keywords → flag for skip
6. Resolve accomplished:
   - For day D: accomplished = day D+1's "yesterday" field (if available)
   - If D+1 doesn't exist: accomplished = day D's "today" field (provisional)
7. Handle skips:
   - Vacation/holiday detected → skip silently
   - User didn't post → skip + warn
   - User-declared vacation days → skip silently
8. Clean up descriptions: strip numbering, join with semicolons
9. Write ./data/standups.json
```

### Vacation/holiday handling

Three mechanisms:

1. **User-declared**: On first run of the month, ask "Any vacation/holiday days this month?" Store answer in `./data/standups.json` under `vacation_confirmed: true`. If user says none, don't ask again. User can set `vacation_confirmed: false` in JSON to re-trigger the question.
2. **Auto-detected**: Replies containing "vacation", "holiday" (case-insensitive) → auto-skip.
3. **Missing days**: User didn't post and day is not declared/detected as vacation → skip + print warning.

## Output Format

### `./data/standups.json`

```json
{
  "extracted_at": "2026-03-31T10:00:00Z",
  "user": "Goran Cabarkapa",
  "user_id": "U028CAVHZ29",
  "channel": "bof-dev-pipe-paid-products",
  "month": "2026-03",
  "vacation_days": [],
  "vacation_confirmed": true,
  "days": [
    {
      "date": "2026-03-03",
      "accomplished": "Send newsletter invite query / Retool app; Admin app: Invite to subscribe tool / createStoryGift mutation",
      "provisional": false
    },
    {
      "date": "2026-03-04",
      "accomplished": "Fix few things on newsletter adverts retool app; AI recommendations",
      "provisional": false
    },
    {
      "date": "2026-03-31",
      "accomplished": "Code review; Deploy staging fixes",
      "provisional": true
    }
  ],
  "warnings": [
    "2026-03-17: No standup reply found"
  ]
}
```

- `provisional: true` — accomplished is based on the day's own "today" (not yet confirmed by next day's "yesterday")
- `provisional: false` — accomplished is confirmed by the next day's "yesterday"
- Days with vacation/holiday or no reply are omitted from the `days` array
- Warnings collected for missing (non-vacation) days

## Skill Structure

```
duck.sh/
├── config.json                    # User configuration
├── data/
│   └── standups.json              # Output
├── skills/
│   └── duck-extract/
│       ├── skill.md               # Skill definition
│       └── instructions.md        # Detailed extraction instructions
└── docs/
    └── superpowers/
        └── specs/
            └── 2026-03-31-slack-standup-extraction-design.md
```

The skill is a Claude Code skill file that instructs the agent on the full extraction workflow, referencing the MCP tools and config.

## Verification

1. Run `/duck-extract` in the duck.sh project directory
2. Confirm interactive setup creates valid `config.json`
3. Verify `data/standups.json` is created with correct structure
4. Spot-check 2-3 days against actual Slack messages
5. Verify vacation keywords are detected and days skipped
6. Verify missing days produce warnings
7. Run again — confirm full overwrite with no stale data
8. Edit `vacation_confirmed: false` in JSON — confirm the question is re-asked on next run
