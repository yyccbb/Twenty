# Twenty developer guide

Twenty is an iOS-first productivity app (web app later). Every calendar event is built from 20-minute slots, start and finish taps snap to that grid, completed events earn tokens that buy badges, and an AI turns long-term goals into slot-sized activities.

This folder is for the people building it. Pick your guide:

| You work on | Read |
|---|---|
| The API, the job worker, the database, deployment | [backend.md](backend.md) |
| Any app that talks to the API | [frontend/README.md](frontend/README.md), then your platform: |
| · the iOS app | [frontend/ios/README.md](frontend/ios/README.md) |
| · the web app (planned) | [frontend/web/README.md](frontend/web/README.md) |

## Where things live

| Thing | Location | Owned by |
|---|---|---|
| Backend design (the spec) | [`../architecture/backend-tech-stack.md`](../architecture/backend-tech-stack.md) | Product. Decisions Q1–Q15 are closed. |
| Shared rules test cases | [`../../contracts/rules/fixtures.json`](../../contracts/rules/fixtures.json) | Product. Every rules implementation must pass them; nobody edits them to make a test pass. |
| Backend code | [`Twenty-backend`](../../../Twenty-backend/README.md) (sibling repo) | Backend |
| API contract | [`Twenty-backend/contracts/openapi.yaml`](../../../Twenty-backend/contracts/openapi.yaml) | Generated from the backend's routes; committed |
| iOS app | not created yet | iOS |
| Web app | not created yet | Web |
| Market research | [`../ideation/market-research.md`](../ideation/market-research.md) | Background only |

> **Spec mismatch:** spec §6 shows `openapi.yaml` under a top-level `contracts/` next to the rules fixtures. In practice it is generated and committed inside the `Twenty-backend` repo (`contracts/openapi.yaml`). Clients generate their code from that copy.

## How the pieces fit

```
                 ../Twenty/contracts/rules/fixtures.json
                  │            │                 │
                  ▼            ▼                 ▼
   Twenty-backend/src/rules   iOS Swift rules   (web imports the TS rules)
         │  (final results)    (previews only)
         ▼
   Twenty-backend routes (Zod) ──npm run openapi──▶ contracts/openapi.yaml
                                                     │               │
                                     swift-openapi-generator   openapi-typescript
                                                     ▼               ▼
                                                 iOS client      web client
```

## The one principle

**The server's result is final; clients preview.** The timing and token rules exist twice: in the backend, which decides, and in each client, which shows an instant preview such as "you'll start at 14:20". Clients send raw device tap times and show what the server returns. If a preview and the server disagree, the server wins, and the disagreement is a bug in the client's rules port.
