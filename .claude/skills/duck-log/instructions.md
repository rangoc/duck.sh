# Duck Log — Step-by-Step Browser Automation Workflow

Follow these steps exactly when `/duck-log` is invoked.

> **Use the direct API (Step 4 below) — do NOT drive the UI form.** The `#started` date field is read via `$('#started').data('date')` at submit time, not from the input's `.val()`. Setting the value via `form_input` or even `$.datepicker('setDate', …)` updates the visible text but leaves `data('date')` undefined, so the submission silently falls back to **today**. This wastes ~30 tool calls per day and creates bogus entries on today's date that then need to be deleted. Skip the UI entirely.

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
4. From config, extract ticket info from `tickets` object. Each ticket has an `id`, `name`, `default_hours`, and `description` field. Also read `special_fridays` for dates with adjusted hour splits. The config is flexible — there may be any number of tickets with any role names.

## Step 1: Determine Days to Log

From `data/standups.json`, collect all days where `provisional` is `false`. Days with `provisional: true` are skipped.

## Step 2: Open duck.dlabs.si

1. Call `mcp__claude-in-chrome__tabs_context_mcp` (with `createIfEmpty: true` if no tab group exists) to get available tabs.
2. Check if duck.dlabs.si is already open. If yes, use that tab. If not, navigate to `https://duck.dlabs.si`.
3. Check each day's logged hours using the calendar titles (see Step 3).

## Step 3: Identify Days Already Logged and Resolve Ticket URIs

### 3a. Read hours per day

Each day cell on the home page has a nested `.info[title="YYYY-MM-DD: Xh"]` tooltip. Extract hours with a single JS call:

```js
fetch('/', { credentials: 'same-origin' })
  .then(r => r.text())
  .then(html => html.match(/title="(2026-04-\d{2}[^"]*)"/g).join('\n'));
```

(Replace the regex with the current month.) Any day showing `>0h` is already logged — skip it. Titles also expose `holiday`, `unpaid-vacation-confirmed`, etc.

If no days need logging, report "All days are already logged" and stop.

### 3b. Resolve ticket URIs (one-time per skill run)

The API needs a full `ticketUri` per ticket, not the Jira key. There are two known URI shapes:

- **Atlassian Jira tickets** (e.g. `BOFTWO-3587`): `jira://d-labs.atlassian.net/<Project Name>/ticket/<id>/<key>`
  - Example: `jira://d-labs.atlassian.net/BoF 2.0/ticket/97976/BOFTWO-3587`
- **Local/internal tickets** (e.g. `INTERNAL-9`): `jira://local/<PROJECT>/ticket/<id>/<key>`
  - Example: `jira://local/INTERNAL/ticket/33851/INTERNAL-9`

You can't construct these blindly (the project name has spaces, capitalisation differs). To resolve one, open the ticket in the UI (click its link on the home page, or set `location.hash` to a guess) and read `document.querySelector('#ticketUri').value`. Do this once per ticket at the start of the run and cache the result.

## Step 4: Submit Entries via the API (do this, not the UI)

### 4a. Endpoint

```
POST /activities/submit
Content-Type: application/x-www-form-urlencoded
Credentials: same-origin (the user's session cookie does the auth)
```

Body fields:

| Field            | Value                                                              |
| ---------------- | ------------------------------------------------------------------ |
| `action`         | `saveActivity` (literal)                                           |
| `ticketUri`      | Resolved URI from Step 3b                                          |
| `time`           | Minutes as a string — `6.5h → "390"`, `1h → "60"`, `2.5h → "150"`  |
| `started`        | Target date as `YYYY-MM-DD`                                        |
| `comment`        | Activity description text                                          |
| `publishComment` | `"0"`                                                              |

A `200` response means the entry was created. The response body is empty on success.

### 4b. Hours → minutes table

Multiply by 60. Common values:

| Hours | Minutes |
| ----- | ------- |
| 1.0   | 60      |
| 2.5   | 150     |
| 4.0   | 240     |
| 6.0   | 360     |
| 6.5   | 390     |
| 7.0   | 420     |
| 7.5   | 450     |
| 8.0   | 480     |

### 4c. Standard vs. special days

For each day in the candidate list, build one entry per ticket in `config.tickets`:

- **Standard day:** use each ticket's `default_hours`. Skip tickets with `default_hours: 0`.
- **Special Friday** (date in `config.special_fridays.dates`): override each ticket's hours with the value in `config.special_fridays.adjustments`. A ticket may only appear in `adjustments` (e.g. Education) — include it only on special Fridays.

For each entry's `comment`:
- `description: "standup"` → use `day.accomplished` from `standups.json`
- otherwise → use the `description` string verbatim (e.g. `"Dailies"`, `"Education"`)

### 4d. Batch submission template

Execute inside `mcp__claude-in-chrome__javascript_tool`. Define a helper once, then fire entries in parallel via `Promise.all`:

```js
window.__submitEntry = async function(e) {
  const body = new URLSearchParams({
    action: 'saveActivity',
    ticketUri: e[1],
    time: String(e[2]),
    started: e[0],
    comment: e[3],
    publishComment: '0',
  });
  const r = await fetch('/activities/submit', {
    method: 'POST',
    credentials: 'same-origin',
    headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
    body: body.toString(),
  });
  return `${e[0]} ${e[3].substring(0, 20)} -> ${r.status}`;
};

const DEV = 'jira://d-labs.atlassian.net/BoF 2.0/ticket/97976/BOFTWO-3587';
const DLY = 'jira://d-labs.atlassian.net/BoF 2.0/ticket/97978/BOFTWO-3589';
const EDU = 'jira://local/INTERNAL/ticket/33851/INTERNAL-9';

const entries = [
  ['2026-04-02', DEV, 390, 'Admin app setup configuration'],
  ['2026-04-02', DLY,  60, 'Dailies'],
  ['2026-04-03', DEV, 240, 'Paywalls new intro offer 25 experiment'],
  ['2026-04-03', DLY,  60, 'Dailies'],
  ['2026-04-03', EDU, 150, 'Education'],
];
Promise.all(entries.map(e => window.__submitEntry(e))).then(rs => rs.join('\n'));
```

Keep each batch ≤ ~8 entries to stay under the tool's output budget and to make failures easy to re-run.

### 4e. Verify after submitting

After each batch (or at the very end) re-read the calendar titles (same fetch as Step 3a) and confirm expected hour totals. Standard day → `7.5h`, special Friday → `7.5h` (4 + 1 + 2.5).

## Step 5: Deleting Mistakes

If an entry lands on the wrong date (e.g. you forgot the `started` field), it's usually logged against **today**. To find and delete it:

1. Navigate the current tab back to the ticket page (e.g. click the ticket link on the home view).
2. The ticket page renders a recent-activity list. Each entry has `.activity > .toolbox > a.delete` with `href="/activities/delete?activityId=<N>"`.
3. Locate your bogus entry (match on the `comment` text and amount) and read its `activityId` from the delete link.
4. Delete via fetch to avoid any confirm dialog:
   ```js
   fetch('/activities/delete?activityId=567167', { credentials: 'same-origin' }).then(r => r.status);
   ```

`200` means deleted. Re-check the calendar to confirm the hours dropped.

## Step 6: Report

After all entries are added, print a summary:
- Number of days logged
- Number of entries added
- Any days that were skipped (already logged / provisional)
- Remind the user to verify on duck.dlabs.si

Example:
```
Logged 13 days on duck.dlabs.si:
- 28 entries added
- 2 special Fridays (3 entries each)
- 1 provisional day skipped
- 3 vacation days skipped
Verify at: https://duck.dlabs.si
```

## Error Handling

- **Browser extension disconnects:** inform the user and stop. They can re-run the skill — already-logged days will be skipped in Step 3a.
- **`fetch` returns non-200:** capture the response body and show it. Usually a permission / session issue.
- **Ticket URI can't be resolved:** fall back to the UI search (type the Jira key into `Find a ticket…`, click the match, then read `#ticketUri`). Cache it and continue.
- **Entry landed on today instead of the target date:** follow Step 5 to delete; verify you're setting `started` in the POST body, not just the input value.
