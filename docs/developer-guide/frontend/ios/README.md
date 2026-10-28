# iOS developer guide

Read [the shared frontend guide](../README.md) first; this page covers only what is specific to iOS.

> **Status:** the iOS app doesn't exist yet. Anything here not fixed by the backend or [the rules](../../rules.md) is a recommendation, marked as such, not a description of existing code.

## 1. Stack

| Concern | Choice | Why |
|---|---|---|
| Language / UI | Swift, SwiftUI (recommended) | |
| API client | Apple's `swift-openapi-generator`, fed by `Twenty-backend/contracts/openapi.yaml` | The yaml is generated from the backend's routes, so the client can't drift from the server |
| Sign-in | Sign in with Apple (`AuthenticationServices`) or email + password | Both work in China. Email also covers users without an Apple ID. Phone/SMS may follow (§7) |
| Notifications | Local notifications (`UserNotifications`); no push in the MVP | The backend sends no push notifications in the MVP, so the app schedules its own |
| Rules previews | A Swift port of `src/rules`, tested against `fixtures.json` | Shows results instantly ("you'll start at 14:20"); the server's result is final |

## 2. Generating the API client

- Copy, or reference with a git submodule, `contracts/openapi.yaml` from `Twenty-backend` into the app's API target, next to `openapi-generator-config.yaml` (generate `types` and `client`).
- Use the build plugin so the client regenerates on every build. A backend API change then shows up as compile errors.
- Add the `Authorization` header and refresh-on-401 logic in one `ClientMiddleware`. That middleware is also the single place where refresh happens (see the shared guide, §3, on why refresh must be single-flight). An `actor` holding the token state works well.
- Add `Idempotency-Key` per user action, not per HTTP attempt. Create it where the action is queued.

## 3. Sign in with Apple

1. Generate a random raw nonce and keep it.
2. Set `request.nonce = sha256(rawNonce)` (hex) on the `ASAuthorizationAppleIDRequest`.
3. Send the ID token **and the raw nonce** to `POST /v1/auth/apple`, plus `TimeZone.current.identifier` as `timezone`, and the user's name as `display_name`. Apple only provides the name on the first authorization, so send it then.
4. Store the access and refresh tokens in the **Keychain**.
5. The bundle ID must be listed in the backend's `APPLE_CLIENT_IDS`, or every sign-in fails with `invalid_apple_token`.

## 4. Local notifications

The app schedules every notification itself. Each needs to be rescheduled when the data behind it changes:

| Notification | Scheduled from | Reschedule when |
|---|---|---|
| Planning prompt (e.g. nightly "plan tomorrow") | `settings.planning_prompts` (ISO weekday + `local_time`) | Settings change |
| Start prompt | Cadenza `planned_start` | Cadenza created, moved, deleted or switched kind; a cascade moves it (`moved` in the extend response). **Back-to-back:** while the previous Cadenza (ending at this one's start) is in progress, don't schedule this prompt, because that Cadenza's prompt becomes "Finish A and start B". If the previous one is finished before its end, schedule this prompt again at `planned_start` |
| "Finish or extend?" prompt | Cadenza `planned_end` − 5 min | Scheduled on start (an early or late start doesn't move it, because the planned end is fixed); moved on extend; removed on finish. If the next Cadenza starts at this one's end, the prompt switches to "Finish A and start B" at `planned_end` |
| Cue occurrence | `/v1/calendar` occurrences (`at`) | Cue or occurrence changes |

Tuttis get no start or finish prompts.

iOS allows at most **64 pending local notifications** per app, and its built-in repeating triggers can't express cue rules like "every 2nd Tuesday". So schedule each occurrence from `/v1/calendar`, which returns cue repeats already expanded, in a rolling window: the next ~48 hours of Cadenza prompts, cues and planning prompts, capped below 64. Top it up on launch and on background refresh, with jitter.

Example, a late start. A Cadenza is planned 14:00–15:00 and the user taps start at 14:07:12:

```
iOS  ── POST /v1/events/42/start {tapped_at: 14:07:12+08:00}, Idempotency-Key ──▶ API
API  : checks tapped_at against server time (no older than 24 h, no more than 2 min ahead)
API  : snaps the tap (start 14:00, tap 14:07:12) → recorded_start 14:20, late_bars 1
API  ◀── 200 {recorded_start: "14:20", late_bars: 1}
iOS  : shows the shorter block (14:20–15:00); the "Finish or extend?" prompt stays at 14:55,
       because the planned end doesn't move
```

## 5. The Swift rules module

- Keep it a pure Swift package with no UIKit, network or `Date()` calls: inputs in, outputs out, like the backend's `src/rules`.
- Run a test target over `../Twenty/contracts/rules/fixtures.json` for all seven groups. Compare times as instants, and require exact objects with no extra keys.
- Read the constants from the fixtures' `constants`. They are placeholders and will change.
- Use it only for previews. Always display the server's response once it arrives.
- When the fixtures change, the Swift port, the TS rules and the backend all change together.

## 6. Offline and the on-device cache

- Cache the calendar locally (SwiftData or SQLite, recommended) so the app opens instantly and works offline.
- **Taps:** capture `Date()` at the moment of the tap, including a tap on a notification's action button while the app isn't open, then persist `{ event_id, kind, tapped_at, idempotency_key }` to a durable queue *before* sending. Send in order when online; retries reuse the key.
- The server rejects taps older than 24 h or more than 2 min in the future (`tap_too_old`, `tap_in_future`). Drop those and tell the user.
- **Auto-finish while offline:** at planned end + 5 min, mark the Cadenza auto-finished locally and remove its prompt. If a finish tap for it is still in the queue, show "Finish pending sync" instead of the deduction. When the tap syncs, the server replaces the auto-finish and the response has the corrected tokens.
- Extend is never queued: it needs the server's preview.
- Other edits made offline need their `version`. On `version_conflict`, refetch and let the user decide.

## 7. App Store requirements

- **Account deletion** must be available in the app (`DELETE /v1/me`).
- Sign in with Apple must be offered. Email + password sign-in is offered alongside it.
- Phone/SMS sign-in may be added for the China launch (planned for the public beta); keep the auth layer open to another provider.

## 8. Checklist for a new screen that writes data

- [ ] Uses the generated client; no hand-written JSON.
- [ ] Sends one `Idempotency-Key` per user action and reuses it on retry.
- [ ] Sends `version` on edits and handles `version_conflict`.
- [ ] Shows the local rules preview, then the server's result.
- [ ] Handles `cadenza_overlap` by highlighting the `conflicts`.
- [ ] Handles `cadenza_in_progress` by asking the user to finish the Cadenza in `conflicts` first.
- [ ] Reschedules affected local notifications.
