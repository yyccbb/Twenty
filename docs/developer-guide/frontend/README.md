# Frontend developer guide

This applies to every Twenty client. Platform specifics are in [ios/](ios/README.md) and [web/](web/README.md).

The backend's API reference is [`Twenty-backend/contracts/openapi.yaml`](../../../../Twenty-backend/contracts/openapi.yaml). A running server also serves it at `/openapi.json`, and [backend guide §7](../backend.md#7-api) lists every endpoint on one page. The rules behind the endpoints (grid, start/finish, extend, overlap, tokens, cues) are in [rules.md](../rules.md); this guide covers how a client uses the API.

## 1. The client's job

| The client owns | The server owns |
|---|---|
| UI and on-device cache | Accounts, calendar, cues, settings |
| Local notifications (planning, start and finish prompts, cues); there are no push notifications in the MVP | Recorded start and finish times |
| Instant previews from its copy of the rules | Token amounts, balance, badges |
| Queuing taps while offline | AI breakdowns (the API keys never leave the server) |

**The server's result is final.** Show the preview immediately, then replace it with the server's response.

## 2. The API client is generated

- Generate the client from `contracts/openapi.yaml`: `swift-openapi-generator` for iOS, `openapi-typescript` for web. Don't hand-write request or response types.
- When the backend changes a route, it regenerates and commits the yaml. Pull it, regenerate, and fix any compile errors: they are the API changes you need to handle.
- Everything is under `/v1`, REST + JSON. Times are ISO 8601 instants (send them with an offset or `Z`). IDs are UUID strings.

## 3. Auth

There are two ways to sign in. Both return the same session: `access_token`, `expires_in`, `refresh_token` and `refresh_expires_at`.

**Sign in with Apple**

1. Run Sign in with Apple on the device and get the ID token.
2. `POST /v1/auth/apple` with `{ id_token, nonce, display_name?, timezone?, device_label? }`.
   - `nonce` is the **raw** nonce. Apple receives its SHA-256 hash (the iOS convention). The server accepts either form.
   - Send `timezone` (IANA, for example `Asia/Shanghai`) on first sign-in.
   - The response also has `created` (true when the account was just made).

**Email + password**

1. Sign up: `POST /v1/auth/email/signup` with `{ email, password, display_name?, timezone?, device_label? }` returns `201`. If the address is in use: `409 email_taken`.
2. Sign in: `POST /v1/auth/email/signin` with `{ email, password, device_label? }`. A wrong password and an unknown email both return `401 invalid_credentials`, so don't tell the user which one it was.
   - The email is checked for format only. There is no verification email and no password reset yet.
   - Passwords are 8–128 characters. Emails are stored lower-cased.
   - Email accounts and Apple accounts are **separate**, even when the addresses match.

**After sign-in** (both methods)

3. Send `Authorization: Bearer <access_token>` on every `/v1` call except `/v1/auth/*`. Access tokens last **15 minutes**.
4. On `401`, or shortly before expiry, call `POST /v1/auth/refresh` with `{ refresh_token }`. The response contains a **new** refresh token: store it and discard the old one.
5. **Refresh from exactly one place.** Each refresh token works once. If two refreshes with the same token run at the same time, the server treats it as token theft and revokes **every** session for the account. Put refresh behind a single-flight lock, so concurrent 401s wait for one refresh.
6. Log out with `POST /v1/auth/logout { refresh_token }`.
7. Delete the account with `DELETE /v1/me`. App Store rules require this for apps with sign-in, so it must be reachable in the UI. It erases personal data. Signing in again creates a fresh account.

Store tokens in secure storage (Keychain on iOS), never in plain preferences.

## 4. Errors

Every error has one shape:

```json
{ "error": { "code": "cadenza_overlap", "message": "…", "conflicts": ["<event id>"] } }
```

Branch on `code`, never on `message` (messages are for logs and may change). Codes worth handling explicitly:

| Code | Status | What the client should do |
|---|---|---|
| `cadenza_overlap` | 409 | A Cadenza would overlap another Cadenza. `conflicts` lists the clashing Cadenza IDs (sorted). Highlight them. |
| `cadenza_in_progress` | 409 | Start sent while another Cadenza is still in progress. `conflicts` holds its ID. Finish it first. |
| `email_taken` | 409 | Email sign-up with an address that already has an account. Offer sign-in instead. |
| `invalid_credentials` | 401 | Email sign-in failed: wrong password or unknown email (deliberately not told apart). |
| `not_cadenza` | 409 | Start, extend or finish was sent for a Tutti. Client bug: only show those actions on Cadenzas. |
| `version_conflict` | 409 | Someone else changed this item. Refetch it, then reapply or ask the user. |
| `bar_misaligned`, `invalid_duration`, `invalid_bars` | 422 | The client sent an off-grid start or a non-whole-bar length. That's a client bug: snap before sending. |
| `start_in_past` | 422 | A Cadenza was created, moved or switched to with a start before now. Offer the next grid mark instead. |
| `too_early` | 422 | Start tapped more than 2 bars before the planned start. |
| `tap_too_old`, `tap_in_future` | 422 | Offline tap older than 24 h, or the device clock is ahead by more than 2 min. Tell the user; drop the queued tap. |
| `window_closed` | 409 | Finish tapped, or extend sent, after planned end + 5 min. The UI shouldn't allow it: check the device clock and drop the queued action. |
| `invalid_state` | 409 | The event is in the wrong state: for example finishing or extending a Cadenza that isn't in progress (or a second finish tap), or deleting or switching the kind of a Cadenza that isn't planned (started, finished or missed). Refetch the event. |
| `unsupported_timezone` | 422 | The zone isn't a whole number of hours from UTC (India, Nepal, …). These zones aren't supported yet. |
| `invalid_timezone` | 422 | Not an IANA zone name. |
| `daily_limit_reached` | 429 | AI breakdown limit for today (5 by default, counted per local day). |
| `insufficient_tokens`, `already_owned` | 409 | Badge redemption failed. |
| `idempotency_key_reused` | 422 | The same `Idempotency-Key` was sent with a different body. Client bug. |
| `idempotency_in_progress` | 409 | The first request with this key is still running. Retry shortly. |
| `unauthorized`, `invalid_token` | 401 | Refresh once; if that fails, sign out. |
| `invalid_request` | 400 | Request failed validation; the message names the field. |

## 5. Writes: Idempotency-Key and versions

- **Send `Idempotency-Key` on every POST/PUT/PATCH/DELETE.** Generate one new UUID per *user action*, store it with the queued request, and reuse it for every retry of that action. The server replays the first response, marked with the `Idempotent-Replayed: true` header. A start or finish therefore never applies twice, and rewards are never paid twice.
- **Optimistic locking:** events and cues carry `version`. Send the version you last saw in `PATCH` bodies. A stale version returns `409 version_conflict`.
- Reads (`GET`) return an `ETag`. Send `If-None-Match` on refreshes to get a cheap `304`.
- Lists use cursor pagination: pass `limit` (1–200, default 50) and the previous response's `next_cursor`, until it is `null`.

## 6. Recording: start, extend, finish

The heart of the app. Only **Cadenzas** are recorded; Tuttis have no start, extend or finish. [Rules §2–§5](../rules.md#2-snapping-one-window-per-grid-mark) has the exact rules, with worked examples; in short:

- **Send the raw tap time.** `POST /v1/events/{id}/start` and `/finish` take `{ tapped_at }`: the device time of the tap. Never send a snapped or computed time. The server snaps it to the 20-minute grid.
- **Start:** the response has `outcome` (`early` / `on_time` / `late` / `missed`), `recorded_start`, `early_bars` and `late_bars`. A late start shortens the Cadenza, because the planned end doesn't move, so the "Finish or extend?" prompt (at planned end − 5 min) doesn't move either.
- **Extend:** only a Cadenza in progress can be extended: at any time during it, or from the "Finish or extend?" prompt until planned end + 5 min. `POST /v1/events/{id}/extend { bars, dry_run }`. It needs the server's preview, so it works online only and is never queued. Call it with `dry_run: true` first to preview. The response lists every Cadenza the cascade would push (`moved`) and any `day_end_warnings`. Each moves only as far as needed, so gaps are used up first, and Tuttis never move. Confirm with the user, then send it again with `dry_run: false`. Update every moved event locally from `moved`.
- **One at a time:** a start fails with `cadenza_in_progress` while another Cadenza is in progress. For back-to-back Cadenzas, the end prompt of A becomes "Finish A and start B" from A's end until end + 5 min; send A's finish, then B's start, with the same `tapped_at`. If the user does nothing by A's end + 5 min, A is auto-finished and B's start is late by at least one bar ([rules §3](../rules.md#3-starting-a-cadenza)).
- **Finish:** the response has `finish_kind` (`on_time` / `shortened` / `abandoned`), `recorded_end`, the token breakdown, `tokens_awarded` and the new `token_balance`.
- **Auto-finish:** at planned end + 5 min the "Finish or extend?" prompt goes away and the server finishes the Cadenza itself (`finish_kind: auto`, no tap involved), deducting one bar of reward. Mark it auto-finished locally at the same moment; your next calendar refresh confirms it.
- **A late-arriving finish replaces the auto-finish, once.** A finish tap made by planned end + 5 min but sent later (offline) is still accepted. The server recomputes the finish from it and corrects the tokens either way. The finish response then reflects the tap. Until it syncs, show "Finish pending sync" instead of the deduction; after, tell the user the change (e.g. "+10 tokens"). A tap after planned end + 5 min gets `409 window_closed`; a second finish tap gets `409 invalid_state` ([rules §4](../rules.md#4-finishing-a-cadenza)).
- **Offline:** queue start and finish taps with their original `tapped_at` and idempotency key, and send them in order once back online (for "Finish A and start B": A's finish first). The server accepts taps up to 24 h old.

## 7. The rules port (previews)

Each native client keeps its own copy of the backend's `src/rules`, used only for instant previews such as "you'll start at 14:20". It must pass every case in [`contracts/rules/fixtures.json`](../../../contracts/rules/fixtures.json):

- **Groups:** `snap`, `start`, `finish`, `tokens`, `extend`, `create`, `timezone`.
- **Matching:** expected objects match exactly, with no extra keys. Compare times as instants, not strings.
- **Constants:** come from the fixture file's `constants`. They are placeholders until the game design is final, so never hard-code them in UI copy.
- **Don't edit the fixtures to make a test pass.** If you think a fixture is wrong, raise it: the backend and every client must change together.

The grid math in brief: bar = 20 min, grid marks at :00/:20/:40, every tap snaps to the mark `g` with `g − 15 min < t ≤ g + 5 min`, except that a finish is on time only within 5 minutes of the planned end (finishing more than 5 minutes early loses the last bar), and a Cadenza may not overlap another Cadenza, while Tuttis overlap freely (intervals are half-open). Everything runs in UTC, which is correct because only whole-hour time zones are supported.

## 8. Time zones and settings

- `GET /v1/me` returns `timezone`, `timezone_supported`, `token_balance` and `settings` (`planning_prompts`, `day_end_local_time`, `preferences`).
- When the device's zone changes, `PATCH /v1/me/settings` with the new `timezone`. If the zone isn't supported, show a clear "not supported yet" state instead of wrong times.
- Planning prompts (`weekday` uses ISO numbering, 1 = Monday, plus `local_time`) are synced settings, but **each device schedules its own local notifications** from them.

## 9. Calendar and cues

- Every event has a `kind`: `cadenza` or `tutti`. Both sit on the grid and last whole bars. Show the start/finish controls and token previews on Cadenzas only.
- A Cadenza can't start in the past: the earliest start is the next grid mark at or after now (`422 start_in_past`). Tuttis can be planned at any time.
- A planned event can switch kind with `PATCH`. Switching to Cadenza can fail with `cadenza_overlap`, or with `start_in_past` if its start has passed.
- `DELETE /v1/events/{id}` removes the event. There is no cancelled state: a deleted event is gone. Only planned events can be deleted. A Cadenza that has started, finished or been missed can't be deleted (`409 invalid_state`), because its record, and any tokens it earned, must stay.
- `GET /v1/calendar?from=&to=` returns events plus **expanded** cue occurrences for the range (at most 62 days). Use it to draw the calendar and to schedule notifications. `GET /v1/events` lists events only (at most 92 days).
- Cues can be at any minute, have no duration and never earn tokens. Repeats use RFC 5545 `RRULE`, which the server expands.
- An occurrence is identified by its **original** time (`original_at`), even after it is moved or snoozed. Its `at` is when it actually fires.
- Mark one occurrence with `PUT /v1/cues/{id}/occurrences/{original_at}` (done / skipped / snoozed / moved).
- Editing or deleting a repeating cue needs a `scope`:
  - `this`: one occurrence; send `occurrence`.
  - `following`: splits the series at `occurrence`; the new part comes back as `created`.
  - `all`: the whole series.

  For `DELETE`, `scope` and `occurrence` go in the query string.

## 10. Goals and AI breakdown

1. `POST /v1/goals/{id}/breakdowns` returns `202 { breakdown_id }`.
2. Poll `GET /v1/breakdowns/{id}` about every 2 s. Status goes `queued` → `running` → `done` or `failed`.
3. Show `items` (each has `duration_bars`, whole 20-minute bars). Let the user pick items and times.
4. `POST /v1/breakdowns/{id}/accept { items: [{ item_id, planned_start, kind?, duration_bars?, title?, notes? }] }` creates all the events in one transaction (`kind` defaults to `cadenza`). If any rule fails (overlap, off-grid), none are created.

## 11. Network etiquette

- **Taps go out immediately.**
- Add **random jitter** to background refreshes, so thousands of devices don't all sync at :00, :20 and :40.
- Pass an `X-Request-Id` header if you have one. The server echoes it back and logs it, so a bug report with the ID can be traced to a server log line.
