# PLAN — grantradar

TDD, one task per focused session. Each task lists files touched, the interfaces it produces, the test to write first, and done-criteria.

## T1 — Supabase schema

Files: `supabase/migrations/0001_init.sql`
Interfaces: tables `grants`, `researcher_profiles`, `draft_aims`; RPC `consume_draft_quota(uuid) returns boolean`.
Test: no vitest — verify via `supabase db push` + two throwaway users; confirm both can read `grants`, neither can read the other's `researcher_profiles`/`draft_aims`.
Done: migration applies cleanly; RLS confirmed; quota RPC not executable by `anon`/`authenticated`.

## T2 — Source normalizers

Files: `src/sources/nih.ts`, `src/sources/nsf.ts`, `src/sources/grantsGov.ts`
Interfaces: `fetchNihRecords(fetchFn?): Promise<RawGrant[]>` (same shape for each), `normalizeGrant(source, raw): GrantRow`.
Test first: `test/sources.test.ts` — fixture JSON per source (captured from a real API response) -> asserts normalized shape, asserts PI email fields are dropped/never mapped into `GrantRow`.
Done: each normalizer round-trips a fixture without throwing; email fields provably excluded from output.

## T3 — CORDIS CSV importer

Files: `scripts/import-cordis-csv.ts`
Interfaces: `parseCordisCsv(buffer): GrantRow[]`.
Test first: `test/cordis-csv.test.ts` — small fixture CSV -> asserts correct row count/shape.
Done: script upserts into `grants` via service-role PostgREST, idempotent on `(source, external_id)`.

## T4 — Indexer + cron entry

Files: `src/indexer.ts`, `src/worker.ts` (`scheduled` export)
Interfaces: `runNightlyIndex(env): Promise<{inserted:number, updated:number, errors:string[]}>`.
Test first: `test/indexer.test.ts` — mocked source fetchers, asserts upsert calls and that a single source failure doesn't abort the other three.
Done: partial-failure tolerant; logs errors per source without crashing the whole run.

## T5 — Supabase data access layer

Files: `src/supa.ts`
Interfaces: `getProfile`, `searchGrants(query, filters)`, `consumeDraftQuota`, `insertDraft`, `updateDraft`, `getDraft` — plain-fetch PostgREST, service-role key, no supabase-js on the Worker.
Test first: `test/supa.test.ts` — mocked fetch asserts REST calls/headers, including the `tsquery` search param shape.
Done: every route in T6/T7 has a tested backing function here.

## T6 — Draft-aims LLM call

Files: `src/llm.ts`
Interfaces: `buildDraftAims(opts: {grant, researchSummary}): Promise<{draftText, alignmentNotes, model, inputTokens, outputTokens}>`.
Test first: `test/llm.test.ts` — mocked Anthropic response, asserts prompt includes both grant abstract and research summary, asserts output shape.
Done: prompt explicitly instructs the model to flag guessed scope; test asserts that instruction is present in the request.

## T7 — Handlers + router + worker entry

Files: `src/handlers.ts`, `src/router.ts`, `src/worker.ts`
Interfaces: `handleMatch`, `handleDraftAims`, `handleGetDraft`, `handleMe`.
Test first: `test/handlers.test.ts`, `test/router.test.ts` — cover quota-exhausted 402, Claude-failure marks draft `failed` with no quota refund, own-row auth via JWT.
Done: every failure path in the LLD's API table has a test.

## T8 — Frontend

Files: `public/index.html`, `public/matches.html`, `public/grant.html`, `public/account.html`, `public/tos.html`, `public/app.js`, `public/styles.css`
Test: manual QA checklist (set up profile, search matches, draft aims, exhaust quota, see ToS disclaimer).
Done: ToS states drafts are not grant-writing advice and PI emails are never surfaced.

## T9 — Deploy + live smoke + launch checklist

Files: `wrangler.jsonc`, `scripts/smoke.ts`
Test: `scripts/smoke.ts` runs against the deployed Worker end-to-end (search for a known grant, draft aims, confirm quota decremented) — this is the ship gate.
Done: `wrangler deploy` succeeds; smoke passes; cron trigger confirmed firing (check first nightly run's `indexed_at` timestamps); secrets set (`ANTHROPIC_API_KEY`, `SUPABASE_SERVICE_ROLE_KEY`); CORDIS CSV imported at least once; pricing page + disclaimers live.
