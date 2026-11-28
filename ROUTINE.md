# ROUTINE.md — instructions for the Super Duper Calendar routine

You keep a personal calendar current. The owner writes plain-language requests;
you turn each one into a job, research it, and write calendar events. A separate
GitHub Action mirrors the events to the owner's phone; you never deal with
notifications.

Requests can be about anything. Never assume a topic, site or API that the
request doesn't point to.

## Files

| Path | Who writes it | What it is |
|---|---|---|
| `requests/<job-id>.md` | the app | the owner's raw request text |
| `jobs/<job-id>.md` | **you** | your interpretation and the job's status |
| `events/<job-id>.json` | **you** | the events this job produced |
| `manual.json`, `overrides.json`, `cancelled.json` | the app | hand-made events, the owner's edits, cancellations |
| `ROUTINE.md`, `schema_version`, `.github/` | the owner | setup |

**Only ever write `jobs/` and `events/`.** Never edit any other file, even to
fix something that looks wrong.

## Every run

1. **Sync.** Make sure the working copy is on `main` and up to date
   (`git fetch origin && git checkout main && git reset --hard origin/main`).
   Record the starting state: `git worktree add ../prev HEAD`.
2. **Reconcile jobs.** Compare `requests/` with `jobs/`:
   - a request with no job file → **new**: interpret it (see "Interpreting");
   - a request whose `sha256sum` differs from the job's `request_hash` →
     **edited**: re-interpret it, keeping the same job ID;
   - a job file with no request file → **deleted**: remove `jobs/<job-id>.md`
     and `events/<job-id>.json`.
3. **Run due jobs** (see "When a job is due") by following each job's own
   `## Interpretation`. Prefer official structured APIs over scraping pages; use
   web search when no API exists. A job that fails records its error (see
   "Job file") and doesn't stop the others.
4. **Write events** following "Events" below, and update `last_run` and
   `last_error` on every job you ran.
5. **Validate.** If you can download the validator, do so and fix every `ERROR`
   line before committing:
   ```sh
   curl -fsSL -o /tmp/syncer https://github.com/Triplem5ds/super-duper-calendar/releases/download/v0.2.0/syncer-linux-amd64
   chmod +x /tmp/syncer && /tmp/syncer validate -data . -prev ../prev
   ```
   If the download fails, skip this step and say so in the commit message. The
   mirror runs the same validation before syncing.
6. **Commit once and push.** If nothing changed, don't commit. Otherwise
   commit with a one-line summary, then: `git fetch origin && git rebase
   origin/main && git push origin main`. If the push is rejected, repeat
   fetch, rebase and push up to 3 times. The files you write are never written
   by anyone else, so the rebase can't conflict.

## Interpreting a request

Decide, and write into the job file:

- **What to look for.** Restate the request precisely.
- **Sources.** Which official API, feed or site answers it; web search if
  nothing structured exists. Prefer sources that give each item a stable ID.
- **`source-key` scheme.** How each event's ID is formed (see "Events"): the
  source's own stable ID for each item where one exists, otherwise a short slug
  you choose now and keep forever.
- **Horizon.** How far ahead to look (from the request; otherwise choose a
  sensible default and list it under `assumptions`).
- **Cadence.** `daily`, `weekly`, or `once` (once = a fixed set of events, such
  as a recurring reminder, that needs no refreshing).
- **Default reminders.** From the request; otherwise a sensible default listed
  under `assumptions`.

Never stop to ask the owner a question. When a request is ambiguous, make your
best guess and list it under `assumptions`. The owner corrects you by editing
the request.

## Job file

```markdown
---
id: <job-id>                 # same as the file name
status: active               # active | done | failed
cadence: weekly              # daily | weekly | once
request_hash: <hex sha256 of requests/<job-id>.md>
last_run: 2026-11-27T06:00:00+02:00
last_error: null             # or a one-line description of the last failure
assumptions:
  - "..."
---
## Interpretation
Look for: ...
Sources: ...
source-key: ...
Horizon: ... Cadence: ...
Output: ...
```

- `status: failed` only when the request can't be interpreted at all. If a run
  fails (source down, nothing found yet), keep `status: active`, set
  `last_error`, and try again on the next due run.
- `status: done` when a one-shot job is fully satisfied (e.g. the thing it
  watched for has happened).

## When a job is due

Use calendar dates in Africa/Cairo. A job is due when it is new or edited, or:

- `daily` and `last_run`'s date is before today;
- `weekly` and `last_run`'s date is 6 or more days before today;
- `once`: only when new or edited.

## Events

`events/<job-id>.json` is a JSON array of:

```json
{
  "id": "<job-id>:<source-key>",
  "job_id": "<job-id>",
  "title": "Short title",
  "start": "2026-11-29T16:35:00+02:00",
  "end": "2026-11-29T18:35:00+02:00",
  "all_day": false,
  "rrule": null,
  "tz": null,
  "reminders": [30],
  "url": "",
  "notes": ""
}
```

- **IDs never change for the same real-world item.** The owner's cancels and
  edits are attached to IDs, so a changed ID brings back an event the owner
  cancelled. Before adding an event, look in the existing file for the same item
  and reuse its ID.
- **Merge, don't replace.** Keep every event whose start is in the past. Update
  or add future events. Remove a future event only when the source no longer
  lists it.
- Timed events use RFC 3339 with an offset. All-day events use `"YYYY-MM-DD"`
  for `start` and `end`, and `end` is the day after the last day (exclusive).
- Recurring events set `rrule` (RFC 5545, e.g. `"FREQ=DAILY"`) and `tz` (e.g.
  `"Africa/Cairo"`); `start`/`end` give the first occurrence.
- `reminders` are minutes before start: at most 5, each at most 40320.
- **A day-long item that needs a reminder at a time of day** (e.g. "remind me at
  9am on the day") is written as a **timed** event at that time (09:00-09:15)
  with `"reminders": [0]`, not as an all-day event: phone reminders on all-day
  events count back from midnight and would ring at the wrong time.
- You may skip researching items whose ID appears in `cancelled.json`; the
  calendar hides them anyway.
