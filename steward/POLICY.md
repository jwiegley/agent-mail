# Steward policy

The thresholds and self-caps a topic steward works by, one row for each key. They live here and nowhere else: the steward reads this file at the start of every turn, and its instructions refer to the rows by key. John edits the copy in the conductor's directory, not the one in the checkout; a change takes effect at the steward's next turn.

`steward-signals` refuses a file that lacks a key, lists a key twice, has a key it does not know, or gives a key a value it cannot use, and it names the key. Nothing falls back to a default, so keep every row and spell each key exactly. Every value is a number above zero, except the two summary rows: summary_local_time is a time of day as HH:MM, and summary_tz is a tz database name.

| key | value | meaning |
|---|---|---|
| stale_min | 45 | Minutes an open item may go without an UPDATE, while its owner is waiting or idle, before the steward sends one NUDGE |
| idle_min | 20 | Minutes a member with no open item may sit waiting or idle before the steward OFFERs it unclaimed BACKLOG items |
| question_min | 30 | Minutes a QUESTION post may go unanswered before the steward routes it to the likeliest member or opens a question for John |
| nudge_gap_min | 30 | Least minutes between two NUDGEs about the same item |
| nudges_per_item_day | 2 | Most NUDGEs about one item in a day; the second says it is the last |
| busy_stuck_h | 2 | Hours a busy member's open item may go without an UPDATE before the steward sends one NUDGE |
| deaf_min | 15 | Minutes a NUDGE may stay unread in the shared folder, with its member waiting, before the member is listed as deaf |
| offer_items | 3 | Most unclaimed BACKLOG items named in one OFFER |
| offer_every_h | 2 | Least hours between two OFFERs to the same member |
| sends_per_turn | 6 | Most mails and posts the steward sends in one turn; more means a loop or a flood, so it stops and asks John |
| sends_per_hour | 20 | Most mails and posts the steward sends in one hour |
| ledger_kb | 8 | Kilobytes LEDGER.md is kept under; finished rows move to done.md |
| log_days | 14 | Days a daily file in log/ is kept |
| back_after_min | 30 | Minutes away after which the steward posts one STEWARD back line when it returns |
| doorbell_delay_min | 5 | Minutes the steward waits before restarting its doorbell while items are active, so a burst of posts is one turn |
| tick_active_min | 15 | Minutes between ticks while an item is active and its owner is not waiting on John |
| tick_idle_min | 60 | Minutes between ticks otherwise |
| summary_local_time | 07:30 | Local time, on a 24-hour clock, at or after which the first turn of the day posts the one-paragraph SUMMARY |
| summary_tz | America/Los_Angeles | Time zone of summary_local_time, as a tz database name |
