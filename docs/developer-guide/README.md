# Twenty developer guide

Twenty is an iOS-first productivity app (web app later). Every calendar event is built from 20-minute bars and is one of two kinds. A **Cadenza** is focused work: the app prompts the user to start and finish it, the taps snap to the grid, and completing it earns tokens that buy badges. Cadenzas never overlap each other. A **Tutti** is an ordinary calendar event that can overlap anything. An AI turns long-term goals into bar-sized activities.

This folder is for the people building it. Pick your guide:

| You work on | Read |
|---|---|
| Anything: the product rules (grid, start/finish, extend, overlap, tokens, cues) | [rules.md](rules.md) |
| The API, the job worker, the database, deployment | [backend.md](backend.md) |
| Any app that talks to the API | [frontend/README.md](frontend/README.md), then your platform: |
| · the iOS app | [frontend/ios/README.md](frontend/ios/README.md) |
| · the web app (planned) | [frontend/web/README.md](frontend/web/README.md) |

## Where things live

| Thing | Location | Owned by |
|---|---|---|
| Product rules | [`rules.md`](rules.md) | Product |
| Shared rules test cases | [`../../contracts/rules/fixtures.json`](../../contracts/rules/fixtures.json) | Product. Every rules implementation must pass them; nobody edits them to make a test pass. |
| Backend code | [`Twenty-backend`](../../../Twenty-backend/README.md) (sibling repo) | Backend |
| API contract | [`Twenty-backend/contracts/openapi.yaml`](../../../Twenty-backend/contracts/openapi.yaml) | Generated from the backend's routes and committed there. Clients generate their code from it. |
| iOS app | not created yet | iOS |
| Web app | not created yet | Web |
| Market research | [`../ideation/market-research.md`](../ideation/market-research.md) | Background only |

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
