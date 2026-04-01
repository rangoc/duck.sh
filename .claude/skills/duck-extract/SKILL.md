---
name: duck-extract
description: Extract daily standup replies from Slack for the current month. Use when the user wants to pull their standup data, extract accomplishments, or prepare data for hour logging.
user-invocable: true
allowed-tools: Read, Write, Edit, Bash(mkdir *), Bash(date *), Bash(ls *), mcp__claude_ai_Slack__slack_read_user_profile, mcp__claude_ai_Slack__slack_search_channels, mcp__claude_ai_Slack__slack_read_channel, mcp__claude_ai_Slack__slack_read_thread, mcp__claude_ai_Slack__slack_search_public_and_private
---

# Duck Extract — Slack Standup Extractor

Extracts your standup replies from a Slack channel's "Team Standup" bot threads, resolves what was actually accomplished each day, and writes structured JSON to `data/standups.json`.

## How It Works

1. Reads `config.json` for channel and bot settings (runs interactive setup if missing)
2. Finds the Team Standup bot's daily messages in your channel for the current month
3. Reads each thread and extracts your replies
4. Parses each reply into: yesterday (item 1), today (item 2), blockers (item 3)
5. Resolves "accomplished" per day: uses next day's "yesterday" as the source of truth
6. Skips vacation/holiday days and warns about missing days
7. Writes `data/standups.json`

## Follow the detailed instructions

Read and follow the step-by-step workflow in [instructions.md](instructions.md).
