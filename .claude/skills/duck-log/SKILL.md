---
name: duck-log
description: Log hours on duck.dlabs.si using standup data from data/standups.json. Use when the user wants to submit their daily hours, log time on duck, or fill in their timesheet.
user-invocable: true
allowed-tools: Read, Write, Edit, Bash(date *), Bash(ls *), mcp__claude-in-chrome__tabs_context_mcp, mcp__claude-in-chrome__tabs_create_mcp, mcp__claude-in-chrome__navigate, mcp__claude-in-chrome__read_page, mcp__claude-in-chrome__computer, mcp__claude-in-chrome__form_input, mcp__claude-in-chrome__find
---

# Duck Log — Hour Logging on duck.dlabs.si

Reads `data/standups.json` and `config.json`, then uses Chrome browser automation to log hours on duck.dlabs.si for each day that has standup data.

## Prerequisites

- User must be logged into duck.dlabs.si in Chrome
- `data/standups.json` must exist (run `/duck-extract` first)
- `config.json` must exist

## Follow the detailed instructions

Read and follow the step-by-step workflow in [instructions.md](instructions.md).
