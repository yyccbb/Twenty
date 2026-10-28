# Product rules

The timing, token, overlap and cue rules that every part of Twenty follows. The backend's `src/rules` implements them, and its results are final. Each client keeps a port only to show results instantly ("you'll start at 14:20"). Every implementation must pass the shared test cases in [`contracts/rules/fixtures.json`](../../contracts/rules/fixtures.json).

These are product decisions (§9 lists them). Never change a rule in code alone: this page, the fixtures, the backend and every client port change together.

**Two kinds of events.** Every event has a `kind`:

| | **Cadenza** | **Tutti** |
|---|---|---|
| Purpose | Focused work. The app prompts the user to start and finish it and times it. | An ordinary calendar event. |
| Grid | Starts on a grid mark, lasts whole bars | Same |
| Start, extend, finish (§3–§5) | Yes | No. The endpoints return `409 not_cadenza`. |
| Tokens (§7) | Yes | Never |
| Overlap (§6) | Never overlaps another Cadenza | Overlaps anything, without limit |
| Moved by an extension's cascade (§5) | Yes | Never |

The rules in §2–§5 and §7 apply to Cadenzas only.

**Event states.** A Cadenza is `planned` until it starts, `in_progress` until it finishes, then `done`. One that nobody starts becomes `missed` once its planned end has passed (§3). A Tutti is only ever `planned`. There is no cancelled state: deleting an event removes it, along with its change history. A Cadenza that has started or finished can't be deleted (`409 invalid_state`), because its record and tokens must stay. Planned and missed events can be deleted.

## 1. The 20-minute grid and time zones

- One **bar** is 20 minutes. Grid marks are :00, :20 and :40.
- Events of both kinds: the start must sit on a grid mark, and `duration_bars` is a whole number ≥ 1.
- **Supported for now: time zones that are a whole number of hours from UTC** (including their daylight-saving shifts). For these, the local :00 / :20 / :40 marks are also UTC :00 / :20 / :40 marks, so **all grid math runs in UTC**. China (UTC+8), most of Europe, the Americas, Japan, Korea and most of Southeast Asia qualify.
- All times are stored as UTC `timestamptz`. The user's IANA time zone (e.g. `Asia/Shanghai`) is kept for display, day boundaries and the day-end setting (§5).

**Out of scope for now: markets with non-whole-hour offsets.** The app warns users in these zones that they're unsupported, and the server rejects these zones with `422 unsupported_timezone`:

| Market | Offset(s) |
|---|---|
| India, Sri Lanka | +5:30 |
| Nepal | +5:45 |
| Iran | +3:30 |
| Afghanistan | +4:30 |
| Myanmar; Cocos (Keeling) Islands | +6:30 |
| Australia: South Australia, Northern Territory, Broken Hill | +9:30 (+10:30 with daylight saving in SA) |
| Australia: Lord Howe Island | +10:30 / +11 (half-hour daylight saving) |
| Australia: Eucla (unofficial) | +8:45 |
| Canada: Newfoundland and parts of Labrador | −3:30 / −2:30 |
| French Polynesia: Marquesas Islands | −9:30 |
| New Zealand: Chatham Islands | +12:45 / +13:45 |

Supporting them later means doing grid math in local time instead of UTC; the stored data doesn't change.

## 2. Snapping: one window per grid mark

Every snap uses the same window. **A grid mark `g` owns the taps from 15 minutes before it to 5 minutes after it**:

```
snap(t) = the grid mark g with  g − 15 min < t ≤ g + 5 min
```

The 5-minute grace applies everywhere. There is **one exception, at the planned end `E`** (§4): a finish counts as on time only from 5 minutes before `E`. A finish tap in `(E − 15, E − 5]`, which `snap` would put at `E`, records `E − 20` instead, so the last bar only counts if the user stays until 5 minutes before the end.

**Tap times come from the device.** A client sends the raw device time of each tap (`tapped_at`), never a snapped time; the server snaps it. A tap made offline is queued and sent later with its original time. The server accepts a tap up to 24 h old and up to 2 minutes ahead of its own clock, and rejects anything else (`422 tap_too_old`, `422 tap_in_future`).

## 3. Starting a Cadenza

`recorded_start = snap(start tap)`. The **planned end stays fixed**. Starting early is allowed by **up to 2 bars (40 min)**.

| Planned 14:00–15:00, start tap at | Recorded start | Result |
|---|---|---|
| 13:05 or earlier | — | **Rejected**: more than 2 bars early |
| 13:05:01–13:25 | 13:20 | **Early by 2 bars** (block 13:20–15:00) |
| 13:25:01–13:45 | 13:40 | Early by 1 bar |
| 13:45:01–14:05 | 14:00 | **On time** |
| 14:05:01–14:25 | 14:20 | **Late by 1 bar** (block 14:20–15:00) |
| 14:25:01–14:45 | 14:40 | Late by 2 bars |
| after 14:45 | 15:00 = end | **Missed** |

An early start is also **rejected** if the longer block would **overlap another Cadenza** (§6). Tuttis don't matter. The app then asks the user to move things first.

A Cadenza that nobody starts is marked **missed** once its planned end has passed (the backend's `events.close_stale` job, every 20 minutes; Tuttis are left alone). It earns nothing and costs nothing (§7).

**One Cadenza at a time.** A Cadenza can't be started, on time or early, while another Cadenza is still in progress (`409 cadenza_in_progress`, with the in-progress Cadenza's ID). The earlier one must be finished first, by a finish tap, an extend-then-finish, or auto-finish at its `E + 5`.

**Back-to-back Cadenzas.** When A ends at `E` and B starts at `E`:

| When the user acts on A's prompt | What happens |
|---|---|
| Finish tap in `(E − 5, E)` | A finishes on time. B's start prompt appears at `E` as usual. |
| Tap in `[E, E + 5]` | The prompt reads **"Finish A and start B"**. One tap finishes A (on time) and starts B (on time). iOS sends A's finish, then B's start, with the same `tapped_at`. |
| Extend | A is extended and B is pushed (§5). |
| Nothing by `E + 5` | A is **auto-finished** (one bar deducted, §4). B's start window `(E − 15, E + 5]` has closed, so B is **late by at least 1 bar**. |

## 4. Finishing a Cadenza

`E` is the Cadenza's current planned end (after any extension). At `E − 5 min`, when the on-time window `(E − 5, E + 5]` opens, iOS shows a **"Finish or extend?"** prompt. It stays up until `E + 5`.

| What happens | Recorded end | Result |
|---|---|---|
| Finish tap in `(E − 5, E + 5]` | `E` | **On time** |
| Finish tap earlier, e.g. `(E − 35, E − 5]` | `E − 20` (generally `snap(tap)`, except that a tap in `(E − 15, E − 5]` records `E − 20`, not `E`) | **Shortened** by the bars not done. Inferred from the tap; there's no separate "shorten" action. Later events are **not moved**. |
| No finish or extend by `E + 5` | `E` | **Auto-finished** by the app, **one bar of reward deducted** |
| Finish tap snaps to ≤ recorded start | — | **Abandoned**, no reward |

| Started 14:00, planned end 15:00, finish tap at | Recorded end | Result |
|---|---|---|
| 14:05 or earlier | — | **Abandoned** |
| 14:05:01–14:25 | 14:20 | Shortened by 2 bars |
| 14:25:01–14:55 | 14:40 | **Shortened by 1 bar** (14:45:01–14:55 is the end exception) |
| 14:55:01–15:05 | 15:00 | **On time** |
| no finish or extend by 15:05 | 15:00 | **Auto-finished** |

**Auto-finish and late-arriving taps.** At `E + 5` the "Finish or extend?" prompt goes away, so the user can't tap finish or extend any more.

1. **At `E + 5` the server auto-finishes.** A delayed background job (pg-boss `startAfter`, rescheduled whenever `E` changes) marks the Cadenza done with `finish_kind: auto` and pays the reward with one bar deducted straight away. No tap is involved.
2. **A finish tap made by `E + 5` that arrives later replaces it, once.** If the device was offline, its queued finish tap (`tapped_at ≤ E + 5`) is sent at the next contact with the server. The server then recomputes the finish from that tap as if it had arrived in time, and writes the difference as a ledger `adjustment`. The difference can go either way: an on-time tap gives back the deducted bar, and an early tap can pay less than the auto-finish did. iOS tells the user (e.g. "Your 14:58 finish synced: +10 tokens").
3. **Everything else is rejected:**
   - a finish tap with `tapped_at` after `E + 5`: `409 window_closed`. The UI can't produce one, so it's a client bug or a wrong device clock;
   - a second finish tap, or one for a Cadenza finished by a tap: `409 invalid_state`;
   - an extend after `E + 5`: `409 window_closed`. Extend needs the online preview, so it is never queued offline.
4. **Order doesn't matter.** Writes for one user are applied one at a time. If the tap arrives before the job runs, the job finds the Cadenza done and does nothing; if after, the tap replaces the job's result. Both end the same way.
5. **Back-to-back Cadenzas.** An offline "Finish A and start B" queues A's finish, then B's start, with the same `tapped_at`, and they are sent in that order. A's late finish replaces its auto-finish, then B's start succeeds because A is finished. B is only marked missed after its planned end passes.
6. **Trust.** Tap times come from the device. They are bounded by the usual checks (§2: no more than 2 minutes in the future, no older than 24 h). That's accepted for the MVP; shortening the window in which a late tap can replace an auto-finish is the lever if it gets abused.

On the device, iOS also marks the Cadenza auto-finished locally at `E + 5`. If a finish tap is still queued, it shows "Finish pending sync" instead of the deduction. A tap on a notification's action button records its time at the moment of the tap, even if the app isn't open.

## 5. Extending a Cadenza

- The user can extend **at any time during the Cadenza**, or from the "Finish or extend?" prompt (`(E − 5, E + 5]`).
- An extension adds whole bars: `E_new = E + 20 min × n`. The finish window and auto-finish then apply to `E_new`.
- Later **Cadenzas** that the extension now runs into are **pushed later**, and the push **cascades**: if a pushed Cadenza now runs into the Cadenza after it, that one is pushed too, and so on down the line. Tuttis are never moved.
  1. Go through the later Cadenzas in start order, starting from the end `edge = E_new`.
  2. If a Cadenza starts before `edge`, shift it so it **starts exactly at `edge`**, keeping its duration. This is the smallest shift that clears the overlap, so any gap before it is used up first. Its new end becomes the new `edge`, and the next Cadenza is checked against it.
  3. Stop at the first Cadenza that starts at or after `edge`; nothing after it moves.
  4. Everything happens in **one transaction**. The response lists every moved Cadenza so iOS can reschedule their notifications. The app shows a **preview** first (`dry_run`).
  5. A pushed Cadenza is always still planned: no other Cadenza can be started while this one is in progress (§3), and every later one ends after it. So a cascade never moves a started or finished Cadenza.

  Example: A runs 14:00–15:00, B 15:20–16:00, C 16:20–16:40. Extending A by 3 bars to 16:00 pushes B to 16:00–16:40; B now runs into C, so C is pushed to 16:40–17:00.
- **Day end:** the user sets when their day ends (`day_end_local_time`, default 00:00, e.g. 02:00). Pushes past it are **allowed but flagged with a warning** in the preview.

## 6. Overlap

**A Cadenza may never overlap another Cadenza**, so the user only ever has one focused task at a time. Tuttis may overlap Cadenzas and each other, without limit. Cues don't count. Intervals are half-open `[start, end)`: a Cadenza ending at 15:00 doesn't overlap one starting at 15:00.

The rule is checked on create, on edit (including switching an event's kind) and on early start. Separately, only one Cadenza can be in progress at a time (§3). Missed Cadenzas are ignored. Violations return `409 cadenza_overlap` with the conflicting Cadenza IDs (sorted) so the app can ask the user to shift things. Extensions can't break the rule, because the cascade moves every Cadenza in the way (§5).

**Switching kinds:** a planned event can switch between Tutti and Cadenza with `PATCH`. Switching to Cadenza runs the overlap check. A Cadenza that has started, finished or been missed can't switch (`409 invalid_state`).

## 7. Tokens

*Placeholder values until the game design is finalized. The current values are the fixtures' `constants`.*

`v` is the token value of one planned bar. Make it an **even whole number** (e.g. 10) so half-value bars stay whole numbers.

```
tokens = v × planned_bars_completed                # full reward on time; proportional otherwise
       + v × 0.5 × early_bars                      # early start: 50% per bar
       + v × 0.5 × extended_bars_completed         # extension: 50% per bar
       − v × (1 if auto_finished else 0)           # no action by E + 5
tokens = max(tokens, 0)
```

- `planned_bars_completed` counts bars of the **original plan** that were actually done. Late starts and early finishes both reduce it, so the reward is `full reward × completed bars / planned bars`.
- **Starting late costs exactly the reward of the missed bars, with no extra penalty.** A missed Cadenza (never started) earns nothing and costs nothing.
- Unused extension bars cost nothing: extending by 2 then finishing after 1 earns 50% for that one bar.

| Planned 14:00–15:00 (3 bars), v = 10 | Tokens |
|---|---|
| Start on time, finish on time | **30** |
| Start 14:20 (1 late), finish on time | 20 |
| Start 13:40 (1 early), finish on time | 30 + 5 = **35** |
| Start on time, finish at 14:40 (shortened) | 20 |
| Start on time, extend 1 bar, finish 15:20 | 30 + 5 = **35** |
| Start on time, no action by 15:05 (auto-finish) | 30 − 10 = **20** |

- Every parameter (`v`, the 50% rates, the auto-finish deduction) lives in a server-side `reward_rules` config row with a version number, so it can change without an app release. Each ledger entry records the rule version used.
- Tokens are paid into an **append-only ledger**: corrections, such as a late-arriving finish tap (§4), are new `adjustment` entries, never edits.
- Only Cadenzas earn tokens. Tuttis and cues never do.

## 8. Cues

A **cue** is a reminder, a separate kind of item: a title and a time, **no duration, no start/finish, no tokens**. iOS fires a local notification at that time; the user can mark an occurrence done, skip it or snooze it.

- **Any minute** is allowed, not just grid marks.
- **Full calendar-style repeat rules**, stored as an RFC 5545 `RRULE` string (the iCalendar standard Apple Calendar and Google Calendar use), e.g. `FREQ=MONTHLY;BYDAY=2TU` = every 2nd Tuesday. Rules are expanded with the `rrule` npm library on the server.
- **Per-occurrence state** (done, skipped, snoozed, moved) lives in its own table, keyed by the occurrence's original time.
- **Editing a repeating cue** offers the same three scopes as Apple Calendar:
  - **This occurrence:** stored as a per-occurrence override (`moved` / `skipped`) or a changed copy of that one occurrence.
  - **This and following:** the series is **split**. The old rule gets an `UNTIL` just before the chosen occurrence, and a new cue starts there with the edited fields. Both share a `series_id` so the app can show them as one series. Overrides after the split point move to the new cue.
  - **All occurrences:** edit the rule in place and keep overrides whose original time still exists in the new rule.
- Cues are never pushed by cascades and don't count toward the overlap rule.
- **Clients don't expand repeat rules.** `GET /v1/calendar` returns the expanded occurrences, so iOS needs no repeat-rule engine of its own. It schedules notifications from those occurrences in a rolling window ([iOS guide §4](frontend/ios/README.md#4-local-notifications)).

## 9. Decisions

All closed. Each row is a product decision; the sections above have the detail.

| # | Decision | Where |
|---|---|---|
| Q1 | Backend language: **TypeScript (Node + Hono)** | [backend guide §2](backend.md#2-stack) |
| Q2 | Early start allowed **up to 2 bars**, using the shared snapping window | §2, §3 |
| Q3 | Extend any time or from the end prompt; extensions push later **Cadenzas** by the **smallest shift** that clears each overlap (gaps are used up first); Tuttis never move; finishing early is an inferred shorten that moves nothing | §4, §5 |
| Q4 | Offline taps are queued; server accepts device time within 24 h if the clock looks sane | §2 |
| Q5 | Append-only ledger + versioned reward rules | §7 |
| Q6 | **Grace is 5 min everywhere**; one snapping window per mark: 15 min before to 5 min after. Exception at the planned end: a finish is on time only in `(E − 5, E + 5]`; a tap in `(E − 15, E − 5]` loses the last bar | §2, §4 |
| Q7 | No action by `E + 5` → **auto-finish, one bar of reward deducted** | §4 |
| Q8 | Rewards: on time = full; shortened or late = proportional; early and extended bars = **50% each** | §7 |
| Q9 | A Cadenza never overlaps another Cadenza; Tuttis overlap without limit | §6 |
| Q10 | Pushes past a **user-set day end** are allowed with a warning | §5 |
| Q11 | Cues: **any minute**, **full RRULE repeats** | §8 |
| Q12 | Only **whole-hour time zones** for now; others listed as out of scope | §1 |
| Q13 | Late start loses only the missed bars' reward, **no extra penalty** | §7 |
| Q14 | Missed (never started) events **cost nothing** | §7 |
| Q15 | Repeating cues support **this / this and following / all** edits | §8 |
| Q16 | Two kinds of events: **Cadenza** (timed, rewarded) and **Tutti** (ordinary). Both use the grid; only Cadenzas earn tokens; planned events can switch kind | Top of page, §6 |
| Q17 | Names: a 20-minute unit is a **bar**; a reminder is a **cue** | §1, §8 |
| Q18 | **Email + password** sign-up/sign-in alongside Sign in with Apple. MVP checks the email's format only (no verification email, no password reset yet); passwords 8–128 chars. Email and Apple accounts stay **separate**, even when the addresses match | [frontend guide §3](frontend/README.md#3-auth) |
| Q19 | **One Cadenza in progress at a time**: the next can't start until the current one finishes. Back-to-back Cadenzas share a "Finish A and start B" prompt in `[E, E + 5]` | §3 |
| Q20 | Auto-finish happens on the server at `E + 5` with no tap. A finish tap made by `E + 5` that arrives later (offline) **replaces it once**, corrected in the ledger; taps after `E + 5` are rejected | §4 |
