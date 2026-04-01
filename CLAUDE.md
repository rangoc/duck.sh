# duck.sh

Automation tool for extracting daily standup data from Slack and logging hours on duck.dlabs.si.

## Workflow

1. Run `/duck-extract` — pulls your Slack standup replies for the current month into `data/standups.json`
2. Run `/duck-log` — reads `standups.json`, opens duck.dlabs.si in Chrome, and logs hours for each day that isn't already logged

## Project Structure

- `.claude/skills/duck-extract/` — Slack standup extraction skill
- `.claude/skills/duck-log/` — duck.dlabs.si browser automation skill
- `data/standups.json` — extracted standup data (current month, overwritten each run)
- `config.json` — user-specific configuration (not committed)
- `config.example.json` — config template
- `docs/superpowers/specs/` — design specs

## Skills

### `/duck-extract`

Extracts your standup replies from Slack for the current month. Uses Slack MCP tools to find the "Team Standup" bot messages in your configured channel, reads the threads, parses your replies (item 1 = yesterday, item 2 = today, item 3 = blockers), and resolves what was actually accomplished each day. Key insight: next day's "yesterday" is the source of truth for today's "today".

- Outputs `data/standups.json`
- Skips vacation/holiday days (auto-detected or user-declared)
- Warns about missing days
- On first run, walks you through interactive setup to create `config.json`

### `/duck-log`

Logs hours on duck.dlabs.si using data from `data/standups.json` and ticket/hour configuration from `config.json`. Uses Chrome browser automation (user must be logged in). Tickets, descriptions, and hour allocations are all driven by the user's config — nothing is hardcoded.

Safety:
- Skips days already logged on duck (>0h in calendar)
- Skips provisional days (unconfirmed accomplishments)
- Safe to re-run — already-logged days are skipped

## Configuration

Each user creates their own `config.json` based on `config.example.json`. This file contains Slack channel name, bot name, user identity, and ticket mappings. It should not be committed (contains user-specific data).
