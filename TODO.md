# TODO

Places where the code doesn't match the design in `docs/developer-guide` yet.

## Twenty-backend

- [ ] Remove `cancelled` from `events.status`: the `event_status` enum, the `events_tutti_status` check, `EventStatus` in `src/routes/schemas.ts`, and `isActive` in `src/rules/overlap.ts`. Migrate existing cancelled rows by deleting them.
- [ ] `DELETE /v1/events/{id}` deletes the event and its change history instead of cancelling it (`cancelEvent` in `src/services/events.ts`).
- [ ] Reject Cadenzas planned in the past with `422 start_in_past` on create, on a move, and on switching a Tutti to Cadenza (rules.md §1; fixtures `create-12`–`create-18`).
- [ ] Allow only one Cadenza in progress at a time with `409 cadenza_in_progress` (rules.md §3; fixtures `start-17`–`start-21`).
- [ ] Switch AI from Qwen to DeepSeek Flash (latest version) through SiliconFlow: add a SiliconFlow provider in `src/ai/`, replace `qwen.ts` and the `QWEN_*` variables with `SILICONFLOW_API_KEY`, `SILICONFLOW_BASE_URL` and `SILICONFLOW_MODEL` (`src/config.ts`, `.env.example`), and make `siliconflow` the default `AI_PROVIDER`.
- [ ] Repoint the code comments that cite "spec §x" (12 files under `src/`) to rules.md or backend.md.
- [ ] Bump the `vendor/Twenty` submodule (pinned at `289a491`) to the latest `main`.
- [ ] Add CI (`.github/workflows/ci.yml`, backend.md §13).
