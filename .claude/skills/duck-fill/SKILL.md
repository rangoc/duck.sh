---
name: duck-fill
description: Generate data/standups.json for the current month with a generic "Work" entry for every working day — no Slack needed. Use when the user wants to log hours without extracting standups, has no standup data, or wants a quick fill so /duck-log can run.
user-invocable: true
allowed-tools: Read, Write, Edit, Bash(mkdir *), Bash(date *), Bash(ls *), Bash(cal *)
---

# Duck Fill — Standup-Free Data Generator

Produces the exact same `data/standups.json` artifact as `/duck-extract`, but without touching Slack. Every working day gets `accomplished: "Work"`.

## How It Works

1. Reads `config.json` (runs a short setup if missing — no Slack required)
2. Refreshes month-specific settings: vacation days and special Fridays
3. Enumerates working days (Mon–Fri) of the current month, up to and including today
4. Skips weekends and vacation/holiday days
5. Writes `data/standups.json` with `accomplished: "Work"`, `provisional: false` for each day
6. Preserves any existing real entries for the current month (never overwrites extracted text)

Output is drop-in compatible with `/duck-log` — run `/duck-log` next.

## Follow the detailed instructions

Read and follow the step-by-step workflow in [instructions.md](instructions.md).
