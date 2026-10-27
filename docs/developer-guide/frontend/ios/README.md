# iOS developer guide

Read [the shared frontend guide](../README.md) first; this page covers only what is specific to iOS.

> **Status:** the iOS app doesn't exist yet. Anything here not taken from the spec is a recommendation, marked as such, not a description of existing code.

## 1. Stack

| Concern | Choice | Source |
|---|---|---|
| Language / UI | Swift, SwiftUI (recommended) | |
| API client | Apple's `swift-openapi-generator`, fed by `Twenty-backend/contracts/openapi.yaml` | spec §2 |
| Sign-in | Sign in with Apple (`AuthenticationServices`) | spec §2 |
| Notifications | Local notifications (`UserNotifications`); no push in the MVP | spec §1 |
| Rules previews | A Swift port of `src/rules`, tested against `fixtures.json` | spec §3 |

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
| Start prompt | Event `planned_start` | Event created, moved or cancelled; a cascade moves it (`moved` in the extend response) |
| Finish / end prompt | `planned_end` | After start (a late start shortens the event), extend or shorten |
| Reminder occurrence | `/v1/calendar` occurrences (`at`) | Reminder or occurrence changes |

iOS allows at most **64 pending local notifications** per app. Schedule a rolling window (for example the next day or two) and top it up on launch and on background refresh, with jitter.

The spec's "late start" flow (§5.2): the app sends `start` and gets `recorded_start` 14:20, then reschedules the finish notification for the new duration.

## 5. The Swift rules module

- Keep it a pure Swift package with no UIKit, network or `Date()` calls: inputs in, outputs out, like the backend's `src/rules`.
- Run a test target over `../Twenty/contracts/rules/fixtures.json` for all seven groups. Compare times as instants, and require exact objects with no extra keys.
- Read the constants from the fixtures' `constants`. They are placeholders and will change.
- Use it only for previews. Always display the server's response once it arrives.
- When the fixtures change, the Swift port, the TS rules and the backend all change together.

## 6. Offline and the on-device cache

- Cache the calendar locally (SwiftData or SQLite, recommended) so the app opens instantly and works offline.
- **Taps:** capture `Date()` at the moment of the tap, then persist `{ event_id, kind, tapped_at, idempotency_key }` to a durable queue *before* sending. Send in order when online; retries reuse the key.
- The server rejects taps older than 24 h or more than 2 min in the future (`tap_too_old`, `tap_in_future`). Drop those and tell the user.
- Other edits made offline need their `version`. On `version_conflict`, refetch and let the user decide.

## 7. App Store requirements

- **Account deletion** must be available in the app (`DELETE /v1/me`).
- Sign in with Apple must be offered (it is the only sign-in method in the MVP).
- The China launch may add phone/SMS sign-in later (spec §2); keep the auth layer open to a second provider.

## 8. Checklist for a new screen that writes data

- [ ] Uses the generated client; no hand-written JSON.
- [ ] Sends one `Idempotency-Key` per user action and reuses it on retry.
- [ ] Sends `version` on edits and handles `version_conflict`.
- [ ] Shows the local rules preview, then the server's result.
- [ ] Handles `overlap_limit` by highlighting the `conflicts`.
- [ ] Reschedules affected local notifications.
