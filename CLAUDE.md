# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Status

Pre-code: the repo holds only `docs/design.md` (source of truth, Draft v0.2) and `.claude/agents/`. There is no build, lint, or test tooling yet. Planned stack: FastAPI + Postgres backend, Next.js/TypeScript PWA frontend, pytest, Docker Compose, GitHub Actions CI. Add real commands here once M0 (setup) lands.

Read `docs/design.md` before any work. Items marked **[DECIDE]** are open and **[PROPOSED]** are unconfirmed, so don't treat them as settled.

## Architecture (from the design doc)

A personal media ranker built from pairwise "which did you like more?" comparisons (Beli-style), not star ratings.

- **Ranking engine is media-agnostic.** It sees only generic items and comparisons. All type-specific code (movies/TV via TMDB, anime via AniList, games via IGDB) lives behind a provider adapter: `search(query) -> list[ItemDTO]`, `get(external_id) -> ItemDTO`. A new media type must mean one new adapter plus a UI tab, with no engine changes.
- **Placement = bucket, then binary insertion.** The user picks a bucket (2 to 5, locked after setup, stored as `buckets` records). The item is then binary-searched into that bucket's range with either-or answers (A, B, or "too tough", which ties and stops).
- **Two kinds of lists.** Watched is ranked per (user, media_type) with buckets and a display-only 0-10 score: `score = hi - ((r - 0.5)/n) * (hi - lo)`, ties share the average. To-Watch is one combined ranked list across all media types, with no buckets or scores. A "to place" queue holds unranked To-Watch items. In Progress and Dropped are unranked shelves. Finishing an item inserts it fresh into Watched.
- **Comparison log is append-only.** Corrections add comparisons and mark old ones `superseded_by`, so rankings can be replayed. MVP correction is only "Re-place" (remove the item and re-run placement).
- **Genre views are filters** on the base ranking, not separate rankings.
- **Benchmark:** simulated noisy users compare insertion only, plus refinement, plus corrections (Kendall tau/Spearman, questions asked). Report only measured numbers.

## Workflow

1. Never commit to main. Never push to any remote; the author does all pushes. One ticket per branch, one branch per pull request.
2. Work in small steps. Keep each change small enough to read in one sitting.
3. Before writing code for a ticket, show a short plan and wait for approval.
4. Do not make design decisions. If `docs/design.md` does not cover something, ask.
5. After each change, explain in plain language what each changed file does and why.
6. For ranking engine code (placement, ties, scores), explain the approach first. The author writes or re-derives that part.
7. Use the `builder` subagent to implement and the `reviewer` subagent to review before the author reads the diff. (`architect` is available for read-only design options.)
8. Keep a decisions log in `docs/decisions.md`.
