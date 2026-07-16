# CLAUDE.md

This repo is product #26 of Febin's 50-SaaS challenge. Nothing is built yet — `docs/` is the source of truth. Read `docs/LLD.md` then `docs/PLAN.md` and execute tasks in order with TDD.

Stack conventions + the shipped reference implementation live in the private repo `febufenn-cyber/50-saas` (`contract-reviewer/`) — same Worker+Supabase+provider patterns (Anthropic/DeepSeek modules reused), but this product is synchronous `Promise.all` fan-out, not the async VPS dispatch+callback pattern.

The `arena-rank` license, the not-LMArena positioning, and the sync fan-out decision in the LLD are verified — do not re-litigate without evidence.
