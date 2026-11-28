---
id: plan-tomorrow-k3m9
status: active
cadence: once
request_hash: fd6c8bbfc33b9afdd75c1e0ca26004c1d0608c7fe1a8ff5c34b7c9fdb5e91c27
last_run: 2026-11-28T14:35:00+02:00
last_error: null
assumptions:
  - "Time zone is Africa/Cairo (the owner's calendar zone)."
  - "Reminder rings exactly at 9pm (reminders: [0]); event lasts 15 minutes."
  - "Recurs daily starting today, 2026-11-28."
---
## Interpretation
Look for: a daily reminder at 21:00 to plan tomorrow.
Sources: none needed; fixed recurring event.
source-key: `daily-plan`
Horizon: unbounded (rrule). Cadence: once
Output: one recurring timed event, FREQ=DAILY, tz Africa/Cairo, 21:00-21:15, reminder at start.
