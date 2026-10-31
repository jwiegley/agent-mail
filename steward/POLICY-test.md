# Steward policy for day zero

The same rows as POLICY.md, with short thresholds so the day-zero checks on `+steward-test` finish in minutes: stale_min, idle_min, question_min, nudge_gap_min, busy_stuck_h, deaf_min, tick_active_min and tick_idle_min are shorter, and every other value is the same. Use this file as the day-zero conductor's policy, and replace it with a copy of POLICY.md before a real topic starts. The same rules apply: keep every row and spell each key exactly.

| key | value | meaning |
|---|---|---|
| stale_min | 2 | Minutes an open item may go without an UPDATE, while its owner is waiting or idle, before the steward sends one NUDGE |
| idle_min | 1 | Minutes a member with no open item may sit waiting or idle before the steward OFFERs it unclaimed BACKLOG items |
| question_min | 1 | Minutes a QUESTION post may go unanswered before the steward routes it to the likeliest member or opens a question for John |
| nudge_gap_min | 1 | Least minutes between two NUDGEs about the same item |
| nudges_per_item_day | 2 | Most NUDGEs about one item in a day; the second says it is the last |
| busy_stuck_h | 0.1 | Hours a busy member's open item may go without an UPDATE before the steward sends one NUDGE |
| deaf_min | 1 | Minutes a NUDGE may stay unread in the shared folder, with its member waiting, before the member is listed as deaf |
| offer_items | 3 | Most unclaimed BACKLOG items named in one OFFER |
| offer_every_h | 2 | Least hours between two OFFERs to the same member |
| sends_per_turn | 6 | Most mails and posts the steward sends in one turn; more means a loop or a flood, so it stops and asks John |
| sends_per_hour | 20 | Most mails and posts the steward sends in one hour |
| ledger_kb | 8 | Kilobytes LEDGER.md is kept under; finished rows move to done.md |
| log_days | 14 | Days a daily file in log/ is kept |
| back_after_min | 30 | Minutes away after which the steward posts one STEWARD back line when it returns |
| doorbell_delay_min | 5 | Minutes the steward waits before restarting its doorbell while items are active, so a burst of posts is one turn |
| tick_active_min | 1 | Minutes between ticks while an item is active and its owner is not waiting on John |
| tick_idle_min | 2 | Minutes between ticks otherwise |
| summary_local_time | 07:30 | Local time, on a 24-hour clock, at or after which the first turn of the day posts the one-paragraph SUMMARY |
| summary_tz | America/Los_Angeles | Time zone of summary_local_time, as a tz database name |
| price_in | 4.00 | Dollars per million input tokens |
| price_cache_write | 5.00 | Dollars per million cache-write tokens |
| price_cache_read | 0.20 | Dollars per million cache-read tokens |
| price_out | 20.00 | Dollars per million output tokens |
