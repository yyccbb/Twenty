# Web developer guide

Read [the shared frontend guide](../README.md) first.

> **Status:** planned for after the iOS app. Nothing is built yet. The points below come from the spec or are early recommendations, marked as such.

## Stack notes

| Concern | Plan | Source |
|---|---|---|
| Language | TypeScript, the same language as the backend | spec §2 |
| API types | `openapi-typescript` generated from `Twenty-backend/contracts/openapi.yaml` (pair it with a small typed fetch wrapper such as `openapi-fetch`, recommended) | spec §2 |
| Rules previews | Reuse the backend's TypeScript `src/rules` instead of writing a third port. It is pure, with no I/O, so it runs in the browser. Package it, or import it from a shared workspace (recommended). | spec §2 |
| Sign-in | Sign in with Apple JS. Its Services ID must be added to the backend's `APPLE_CLIENT_IDS`. Send the nonce exactly as you gave it to Apple; the server accepts the raw value. | spec §2 |
| Notifications | None in the MVP. Notifications are an iOS feature; web shows the same calendar. | spec §1 |

## Things that differ from iOS

- **Token storage:** don't keep the refresh token in `localStorage`. Prefer memory for the access token, plus a storage approach decided when web auth is designed. The backend currently returns tokens in JSON bodies, not cookies, so web may need backend changes. Raise this before building.
- **Time zone:** read it from `Intl.DateTimeFormat().resolvedOptions().timeZone` and sync it through `PATCH /v1/me/settings` like iOS does. Handle `unsupported_timezone` the same way.
- **Multiple tabs:** refresh must stay single-flight *across tabs* (a `BroadcastChannel` or Web Locks), or two tabs refreshing together will revoke every session.
- **Mainland China:** use `registry.npmmirror.com` for npm installs there, and avoid CDNs that are blocked in China.
