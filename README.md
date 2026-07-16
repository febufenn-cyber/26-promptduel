# promptduel

A blind A/B arena for your own prompts and models, with Elo leaderboards scoped to your own eval sets — a private team tool, not a public model-ranking site.

**Status: planned — not yet built (50-SaaS challenge #26)**

## The problem

Prompt engineers and AI teams tweak a prompt or swap a model and just eyeball the difference — there's no lightweight, blind, repeatable way to A/B their own prompt/model variants against their own test cases and see which one actually wins over time.

## Target buyer

Prompt engineers and small AI teams who need a private eval tool for their own prompt/model variants, not a public leaderboard audience.

## Pricing hypothesis

Rs799/mo flat subscription, with a monthly duel-run quota to bound LLM provider cost.

## Stack summary

Cloudflare Worker (TypeScript) fans out each duel to 2+ provider API calls in parallel via `Promise.all`, reusing the reference's Anthropic/DeepSeek provider modules. Supabase Postgres (RLS, user/team-owned) stores eval sets, contenders, duels, and votes; Elo is computed with the `arena-rank` package.

## How to continue this build

Nothing is implemented yet. Read `docs/LLD.md` for the architecture and data model, then `docs/PLAN.md` for the ordered TDD task list, then follow `CLAUDE.md` for how this repo relates to the shipped reference implementation.

## Risks / Constraints

- `arena-rank` is Apache-2.0 — safe to depend on, pin an exact version.
- **Do not brand or position this as LMArena** or any lookalike of it — this is a private per-team eval tool, not a public general-model leaderboard. Keep eval sets and leaderboards scoped to the owning user/team, never public by default.
- Fan out to provider APIs via `Promise.all` (parallel), not sequential calls — a duel's latency should be roughly the slowest single contender call, not the sum.
- Votes live in Supabase, RLS-scoped to the owning user/team.
- Users may paste sensitive or proprietary prompts into third-party LLM APIs — this must be disclosed, since promptduel does not control what Anthropic/DeepSeek do with that data beyond the reference's existing provider integrations.
