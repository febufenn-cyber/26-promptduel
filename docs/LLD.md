# LLD — promptduel

## Architecture (request flow)

```
Browser (magic-link auth)
   |  POST /api/duels {eval_item_id, contender_a_id, contender_b_id}
   v
Worker (src/handlers.ts)
   |-- consume_duel_credit(user_id)          -> Supabase RPC (monthly quota, see below)
   |-- Promise.all([
   |     callProvider(contenderA),            -> src/providers/anthropic.ts or deepseek.ts
   |     callProvider(contenderB)              (reused/adapted from the reference)
   |   ])
   |-- randomize position (A/B shuffle) so voter can't infer identity from order
   |-- insert duels row (responses stored, winner=null, identities hidden in response)
   v
Browser shows blind {duel_id, prompt, response_left, response_right}   [sync, one round trip]

   |  POST /api/duels/:id/vote {winner: 'left'|'right'|'tie'}
   v
Worker
   |-- record vote, map left/right back to contender_a/contender_b
   |-- updateElo(contenderA, contenderB, result) -> `arena-rank` package
   |-- update elo_ratings rows
   v
Browser reveals which contender was which + updated Elo    [sync]
```

## Data model (Supabase Postgres, RLS on)

```sql
create table public.profiles (
  user_id uuid primary key references auth.users(id) on delete cascade,
  duel_credits int not null default 20,     -- monthly quota, resets on billing period
  quota_period_start date not null default date_trunc('month', now()),
  created_at timestamptz not null default now()
);
alter table public.profiles enable row level security;
create policy "read own profile" on public.profiles for select using (auth.uid() = user_id);

create table public.eval_sets (
  id uuid primary key default gen_random_uuid(),
  user_id uuid not null references auth.users(id) on delete cascade,
  name text not null,
  created_at timestamptz not null default now()
);
alter table public.eval_sets enable row level security;
create policy "read own eval sets" on public.eval_sets for select using (auth.uid() = user_id);

create table public.eval_items (
  id uuid primary key default gen_random_uuid(),
  eval_set_id uuid not null references public.eval_sets(id) on delete cascade,
  prompt_text text not null,
  created_at timestamptz not null default now()
);
alter table public.eval_items enable row level security;
create policy "read own eval items" on public.eval_items for select
  using (eval_set_id in (select id from public.eval_sets where user_id = auth.uid()));

create table public.contenders (
  id uuid primary key default gen_random_uuid(),
  user_id uuid not null references auth.users(id) on delete cascade,
  eval_set_id uuid not null references public.eval_sets(id) on delete cascade,
  label text not null,
  provider text not null check (provider in ('anthropic','deepseek')),
  model text not null,
  system_prompt text,
  created_at timestamptz not null default now()
);
alter table public.contenders enable row level security;
create policy "read own contenders" on public.contenders for select using (auth.uid() = user_id);

create table public.duels (
  id uuid primary key default gen_random_uuid(),
  user_id uuid not null references auth.users(id) on delete cascade,
  eval_item_id uuid not null references public.eval_items(id),
  contender_a_id uuid not null references public.contenders(id),
  contender_b_id uuid not null references public.contenders(id),
  response_a text, response_b text,
  winner text check (winner in ('a','b','tie')),   -- null until voted
  voted_at timestamptz,
  created_at timestamptz not null default now()
);
alter table public.duels enable row level security;
create policy "read own duels" on public.duels for select using (auth.uid() = user_id);

create table public.elo_ratings (
  contender_id uuid primary key references public.contenders(id) on delete cascade,
  eval_set_id uuid not null,
  rating numeric not null default 1500,
  games int not null default 0,
  updated_at timestamptz not null default now()
);
alter table public.elo_ratings enable row level security;
create policy "read own ratings" on public.elo_ratings for select
  using (eval_set_id in (select id from public.eval_sets where user_id = auth.uid()));

create or replace function public.consume_duel_credit(p_user uuid) returns boolean
language plpgsql security definer set search_path = public as $$
declare v_period date := date_trunc('month', now());
begin
  update profiles set duel_credits = 20, quota_period_start = v_period
    where user_id = p_user and quota_period_start < v_period;
  update profiles set duel_credits = duel_credits - 1
    where user_id = p_user and duel_credits > 0;
  return found;
end $$;
revoke execute on function public.consume_duel_credit(uuid) from public, anon, authenticated;
grant execute on function public.consume_duel_credit(uuid) to service_role;
```

Monthly quota (`duel_credits`), not the reference's one-free-credit-then-topup model — flat-subscription product, same pattern as grantradar's `consume_draft_quota`.

## API routes

| Route | Method | Auth | Behavior | Failure modes |
|---|---|---|---|---|
| `/config.js` | GET | none | Public Supabase URL/anon key | — |
| `/api/eval-sets` | POST/GET | Supabase JWT | Create/list eval sets | — |
| `/api/contenders` | POST/GET | Supabase JWT | Create/list contenders per eval set | — |
| `/api/duels` | POST | Supabase JWT | Consumes 1 duel credit, fans out `Promise.all` to 2 contenders, returns blind result | quota exhausted (402), both providers fail -> refund credit + 502, one fails -> that side shows an error, vote disabled |
| `/api/duels/:id/vote` | POST | Supabase JWT (own row) | Records winner, updates Elo via `arena-rank`, reveals identities | already voted (409), not found (404) |
| `/api/leaderboard/:eval_set_id` | GET | Supabase JWT (own set) | Elo ratings sorted, sync DB query | — |
| `/api/me` | GET | Supabase JWT | Profile + remaining duel credits | — |

## LLM strategy

- Providers: reuse the reference's **anthropic** (`claude-sonnet-4-6`) and **deepseek** (`deepseek-chat`) provider modules — a contender is just `{provider, model, system_prompt}`, called with the eval item's `prompt_text` as the user message. Plain text completion, no `json_schema` output — the whole point is comparing raw model output, not extracting structured data.
- Fan-out: both contender calls issued via `Promise.all`, each with its own timeout; a duel's total latency is bounded by the slower of the two calls, not their sum.
- Elo: `arena-rank` (Apache-2.0, pin exact version) computes the rating update from each `{contender_a, contender_b, winner}` result; ratings persist in `elo_ratings`, recomputed incrementally per vote, not batch-recomputed from scratch.
- Cost estimate: 1 duel = 2 parallel completions, ~300-800 tokens combined in/out each depending on the prompt -> roughly Rs1-3/duel-pair. A 20-duel/mo default quota keeps provider spend well under the Rs799/mo price, but **unit economics need monitoring** if contenders use longer system prompts or more providers are added later (flagged in Risks).
- Sync vs async: 2 parallel completions in `Promise.all` finish in low single-digit seconds even on a slow provider — **fully synchronous**, no VPS/callback pattern needed. If a future version adds bulk eval-set batch runs (many items x many contenders in one action), that aggregate call *would* exceed the ~100s cap and should move to a queued/async pattern — that's explicitly why bulk batch runs are out of scope for v1, not an oversight.

## Frontend pages

`index.html` (login), `eval-sets.html` (create eval sets + items), `contenders.html` (configure contenders: provider, model, system prompt), `duel.html` (run a blind duel, vote), `leaderboard.html` (Elo table per eval set), `account.html` (plan/quota), `tos.html` (disclosure: prompts are sent to third-party LLM APIs; don't paste secrets).

## Error handling + quota flow

Duel credit consumed atomically via `consume_duel_credit` before the fan-out call. If **both** contenders fail, the credit is refunded and the duel marked errored (nothing to vote on). If **one** contender fails, the duel is still shown with an error on that side and voting disabled (not a fair blind comparison) — credit is refunded in this case too, since no valid result was produced. No refund path exists for a successfully-completed duel regardless of vote outcome.

## Integrations and launch gates

- Anthropic and DeepSeek API keys — already provisioned and used elsewhere in the portfolio, no new integration work.
- `arena-rank` package: pin an exact version and confirm the Apache-2.0 license on that pinned version before first use (license text can change between majors).
- No OAuth, no other third-party partnerships required for v1.

## Security notes

- RLS: every table is `user_id = auth.uid()` owned or scoped through an owned `eval_set_id`/`contender_id`, matching the reference's single-owner pattern (unlike grantradar/docwhisper, this product has no team-sharing in v1).
- Secrets via `wrangler secret put`: `ANTHROPIC_API_KEY`, `DEEPSEEK_API_KEY`, `SUPABASE_SERVICE_ROLE_KEY`.
- Rate-limit `/api/duels` per user beyond the monthly credit check (e.g. a short per-minute cap) to prevent a burst of rapid-fire duels from spiking concurrent provider calls.
- User-entered `system_prompt` and `prompt_text` are sent verbatim to third-party provider APIs — disclose this plainly in the ToS; do not log full prompt/response content anywhere beyond the `duels` row itself.

## Out of scope for v1

Bring-your-own-API-key support, a public/global leaderboard (explicit non-goal — this is not LMArena), bulk eval-set batch runs (many items x many contenders in one server call), providers beyond Anthropic/DeepSeek, team/multi-user sharing of eval sets, image or multimodal prompts, configurable Elo K-factor per eval set.
