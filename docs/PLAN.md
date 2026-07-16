# PLAN — promptduel

TDD, one task per focused session. Each task lists files touched, the interfaces it produces, the test to write first, and done-criteria.

## T1 — Supabase schema

Files: `supabase/migrations/0001_init.sql`
Interfaces: tables `profiles`, `eval_sets`, `eval_items`, `contenders`, `duels`, `elo_ratings`; RPC `consume_duel_credit(uuid) returns boolean`.
Test: no vitest — verify via `supabase db push` + two throwaway users; confirm neither can read the other's eval sets/contenders/duels/ratings.
Done: migration applies cleanly; RLS confirmed; RPC not executable by `anon`/`authenticated`.

## T2 — Provider call adapters

Files: `src/providers/anthropic.ts`, `src/providers/deepseek.ts`
Interfaces: `callContender(contender: {provider, model, systemPrompt}, promptText: string, fetchFn?): Promise<{text:string, model:string}>` — same signature for both providers, adapted from the reference's provider modules (plain completion, no `json_schema`).
Test first: `test/providers.test.ts` — mocked fetch per provider, asserts correct request shape/auth header, asserts typed error on non-2xx.
Done: both providers implement the identical `callContender` interface so the fan-out code in T4 doesn't branch on provider.

## T3 — Blind pairing + Elo

Files: `src/blind.ts`, `src/elo.ts`
Interfaces: `assignBlindPositions(contenderA, contenderB): {left, right, mapping}` (random shuffle), `updateElo(ratingA, ratingB, result: 'a'|'b'|'tie'): {newRatingA, newRatingB}` (wraps `arena-rank`).
Test first: `test/blind.test.ts` (asserts both orderings occur across many runs, mapping is always recoverable), `test/elo.test.ts` (asserts winner's rating increases, loser's decreases, tie moves both toward each other).
Done: `arena-rank` version pinned in `package.json`; license re-confirmed at that pinned version.

## T4 — Duel dispatch handler

Files: `src/handlers.ts` (`handleCreateDuel`)
Interfaces: `handleCreateDuel(req, env): Promise<Response>` — consumes credit, `Promise.all`s two `callContender` calls, applies blind positions, inserts `duels` row.
Test first: `test/duel-handler.test.ts` — mocked providers: both-succeed, one-fails (error shown, no vote, credit refunded), both-fail (credit refunded, 502).
Done: fan-out is provably parallel in the test (both mocked calls invoked before either resolves), not sequential.

## T5 — Vote handler

Files: `src/handlers.ts` (`handleVote`)
Interfaces: `handleVote(req, env, duelId): Promise<Response>`.
Test first: `test/vote-handler.test.ts` — asserts winner recorded, Elo updated via T3's `updateElo`, identities revealed in the response, double-vote returns 409.
Done: vote path never touches provider APIs (pure DB + Elo math).

## T6 — Supabase data access layer

Files: `src/supa.ts`
Interfaces: `getProfile`, `consumeDuelCredit`, `refundDuelCredit` (best-effort increment, not the reference's strict refund RPC — quota-based, see LLD), `createEvalSet`, `listContenders`, `insertDuel`, `updateDuelVote`, `getLeaderboard`.
Test first: `test/supa.test.ts` — mocked fetch, asserts correct PostgREST REST calls/headers.
Done: every write used by T4/T5 has a tested function here.

## T7 — Router + worker entry

Files: `src/router.ts`, `src/worker.ts`
Interfaces: routes all endpoints in the LLD's API table.
Test first: `test/router.test.ts`.
Done: 404 on unknown routes, method mismatches handled.

## T8 — Frontend

Files: `public/index.html`, `public/eval-sets.html`, `public/contenders.html`, `public/duel.html`, `public/leaderboard.html`, `public/account.html`, `public/tos.html`, `public/app.js`, `public/styles.css`
Test: manual QA checklist (create eval set + item, configure two contenders, run a duel blind, vote, confirm identities reveal + leaderboard updates, exhaust quota).
Done: ToS discloses prompts go to third-party APIs; UI never leaks contender identity before a vote is cast.

## T9 — Deploy + live smoke + launch checklist

Files: `wrangler.jsonc`, `scripts/smoke.ts`
Test: `scripts/smoke.ts` runs against the deployed Worker end-to-end (create an eval set + 2 contenders, run a duel, vote, confirm leaderboard reflects the updated Elo) — this is the ship gate.
Done: `wrangler deploy` succeeds; smoke passes; secrets set (`ANTHROPIC_API_KEY`, `DEEPSEEK_API_KEY`, `SUPABASE_SERVICE_ROLE_KEY`); pricing page + ToS live.
