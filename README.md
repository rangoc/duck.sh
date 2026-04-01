# duck.sh

Automates daily hour logging on [duck.dlabs.si](https://duck.dlabs.si) using your Slack standup replies. Built as a set of [Claude Code](https://docs.anthropic.com/en/docs/claude-code) skills.

## What it does

Every day, your team posts standups in Slack via the "Team Standup" bot. You reply with what you did yesterday, what you'll do today, and any blockers. **duck.sh** reads those replies, figures out what you actually accomplished each day, and logs the hours on duck for you.

Two commands, run in order:

1. **`/duck-extract`** — Pulls your standup replies from Slack for the current month, resolves what was actually accomplished each day, and saves it to `data/standups.json`
2. **`/duck-log`** — Opens duck.dlabs.si in Chrome and logs hours for each day that isn't already logged

That's it. Run both at the end of the month (or whenever) and your timesheet is done.

## How it works

### Extraction (`/duck-extract`)

Connects to Slack via MCP, finds the standup bot's daily threads in your channel, and parses your replies:

```
1. What I did yesterday    ← item 1
2. What I'll do today      ← item 2
3. Blockers                ← item 3
```

The key insight: **next day's "yesterday" is the source of truth for today's "today"**. What you said you'd do is a plan; what you report having done the next morning is what actually happened.

- Auto-detects vacation/holiday keywords in replies
- Lets you declare vacation days upfront (only asks once per month)
- Warns about missing standup days
- Output: `data/standups.json`

### Logging (`/duck-log`)

Reads `standups.json` and your ticket config, then automates Chrome to fill in the duck.dlabs.si activity form:

- Searches for each ticket, fills in the description, sets hours and date, clicks "Add"
- Skips days already logged (checks the calendar bar for >0h)
- Skips provisional days (where accomplishments haven't been confirmed yet)
- Safe to re-run — already-logged days are never touched

## Prerequisites

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) (CLI, desktop app, or IDE extension)
- **Slack MCP** — connected to your workspace ([setup guide](https://modelcontextprotocol.io/integrations/slack))
- **Chrome MCP** (Claude in Chrome) — for browser automation ([install](https://claude.ai/chrome))
- Logged into duck.dlabs.si in Chrome

## Setup

On first run of `/duck-extract`, an interactive setup walks you through:

1. Detects your Slack identity automatically
2. Asks which channel has your standups
3. Asks for the standup bot's name (default: "Team Standup")
4. Asks about your ticket setup — which tickets, IDs, and hour splits
5. Asks about special Fridays (education days, etc.)

This creates a `config.json` in the project root (git-ignored — it contains your personal settings).

You can also copy `config.example.json` and fill it in manually.

## Usage

```
# In Claude Code, from the project directory:

/duck-extract    # Step 1: Pull standup data from Slack
/duck-log        # Step 2: Log hours on duck.dlabs.si
```

Run `/duck-extract` first to get fresh data, then `/duck-log` to submit it. Both are idempotent — safe to re-run anytime.

## Project structure

```
duck.sh/
├── .claude/skills/
│   ├── duck-extract/       # Slack standup extraction skill
│   └── duck-log/           # duck.dlabs.si browser automation skill
├── data/
│   └── standups.json       # Extracted standup data (git-ignored)
├── config.json             # Your personal config (git-ignored)
├── config.example.json     # Config template
└── docs/                   # Design specs
```

## Configuration

`config.json` stores your Slack channel, bot name, user identity, ticket mappings, and special day rules. The ticket setup is flexible — any number of tickets with any hour splits. Example structure in `config.example.json`.

Key fields:
- **tickets** — each ticket has an ID, name, default hours, and a description field (`"standup"` = use accomplished text from standups, anything else = use as-is)
- **special_fridays** — dates where hour splits differ (e.g., education days with reduced dev hours)
