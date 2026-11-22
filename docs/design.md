# Design Doc: Personal Media Ranker

Status: Draft v0.2 | Author: Patricio Leroux | Last updated: 2026-11-18

> Items marked **[DECIDE]** are open choices. Items marked **[PROPOSED]** were drafted by Claude and not yet confirmed by you. This doc only counts once you can defend every line.

## 1. Problem

Star ratings and watchlists are bad at telling you two things: how your favorites really compare, and what to watch next. Ratings drift over time, most things end up 4 stars, and a to-watch list has no order.

This project builds a personal ranking from **pairwise comparisons**, the way Beli does for restaurants:

- For **watched** items, you answer "which did you like more?" when you add something.
- For **to-watch** items, you answer "which do you want to watch first?" so the top of the list is your next pick.

Every ranking comes from the user's own answers. There are no algorithmic recommendations.

## 2. Goals and Non-Goals

### Goals (MVP)
- Support four media types: movies, TV, anime, and video games. Search each and add items to a Watched or To-Watch list.
- Make adding a new media type cheap: one new provider adapter, no changes to the ranking engine (see section 6.7).
- Build the ranking from a few either-or questions each time an item is added, so it grows with the list.
- An In Progress shelf for what you are currently watching, playing, or reading. It is not ranked; when you finish an item you tap "Finish," pick a bucket, and answer the placement questions.
- Let the user correct the ranking: pick two items, re-compare, and have the order update with a few follow-up questions.
- One base ranking per (media type, list). Genre views are filters on that ranking, not separate rankings.
- Show a 0-10 score per item, derived from its position in the ranking (display only).
- User accounts, persistent data, and a deployed, mobile-first web app (installable as a PWA) for personal use and friends.
- A simulation benchmark that measures ranking accuracy and the number of questions asked.

### Non-Goals (later, if time allows)
- Genre and runtime filters, and combining filters (the MVP has only the media type filter on To-Watch; adapters should still return genres and runtime so the data is ready)
- Changing the number of buckets after setup (locked in the MVP; revisit later)
- Genre-only rankings with their own separate comparisons (the user-selectable "divide by genre instead of one base list" mode)
- Friends and recommendations from friends (the first milestone after the core is working; see M7)
- Import from Letterboxd, Goodreads, or Trakt
- Books, music, podcasts, and other media types (the design must make these easy to add later)
- Cross-media ranking (types are ranked separately in the MVP)
- Uncertainty-aware pair selection using Bayesian ranking (MVP uses simple selection)
- Native iOS/Android apps and push notifications. The MVP is a mobile-first web app installable to the home screen; it shows a comparison question when opened.
- Public store listings and any monetization (ads, payments)

## 3. Users and Core Flows

1. **Add item:** search the provider for that media type, add to Watched or To-Watch, then place it in the ranking (section 6.1).
2. **Refinement question:** when the app opens, occasionally show two neighboring items and ask which is better (section 6.3).
3. **View rankings:** the base ranking with scores, plus genre filters. An item with several genres shows up under each matching filter.
4. **Correct a ranking:** select two items that look wrongly ordered, re-compare them, and answer a few follow-up questions (section 6.4).
5. **Start:** move an item to In Progress from the ranked To-Watch list or straight from the "to place" queue, without ranking it first (decided). In Progress items are a simple shelf with no comparisons and no progress notes (decided).
6. **Finish:** from In Progress, tap "Finish," pick a bucket, and the item is inserted into the Watched ranking with the usual questions. It is inserted fresh, because "liked more" is a different question from "want to watch first" **[PROPOSED]**.
7. **Stop partway:** the user can remove the item, or simply leave it in In Progress, or (optionally) move it to an unranked Dropped shelf. Dropped items never enter the ranking. **[PROPOSED reading of "keep a dropped shelf as an option": confirm]**
8. **Re-place:** on any ranked item, tap "Re-place." The item is removed from its list and treated as a brand-new item: pick a bucket (Watched only; the same or a different one) and answer the placement questions again. Its earlier comparisons are marked superseded (decided).

## 4. Stack

| Layer | Choice | Reason |
|---|---|---|
| Backend | Python, FastAPI | Ranking engine and simulation benchmark live in the same language as numpy/scipy; typed, clean REST API |
| Database | Postgres | Relational data; the append-only comparison log can be replayed to recompute rankings |
| Frontend | Next.js + TypeScript (React), mobile-first PWA | One codebase for desktop and phone; installable to the home screen; no app store needed |
| Auth | **[DECIDE]** Clerk / Supabase Auth | Avoids hand-rolling password handling, which is not the interesting part of this project |
| Data sources | TMDB (movies, TV), AniList (anime), IGDB (games) | Free APIs per type; IGDB needs Twitch developer credentials. Each sits behind a provider adapter. |
| Testing | pytest | Standard; used for ranking and API tests |
| Infra | Docker, GitHub Actions | CI on every PR |
| Deploy | **[DECIDE]** Render / Fly.io (AWS later if there is a reason) | Cheap or free to start; pick the one you can actually finish |

## 5. Data Model (draft) **[PROPOSED]**

- **users**: id, auth_provider_id, created_at (credentials are handled by the auth provider)
- **items**: id, media_type (`movie` | `tv` | `anime` | `game` | ...), source (`tmdb` | `anilist` | `igdb` | ...), external_id, title, year, image_url, genres (list of strings from the provider), extra (JSON for type-specific fields). Unique on (source, external_id).
- **user_items**: id, user_id, item_id, list_type (`to_place` | `to_watch` | `in_progress` | `watched`; only `to_watch` and `watched` are ranked), bucket_id (see **buckets**; **[DECIDE]** whether To-Watch uses buckets), position (the item's place in the ordered ranking), tie_group (nullable; items sharing one are tied), added_at
- **buckets**: id, user_id, media_type, name (editable label), order, score_lo, score_hi. Created once at setup (see 6.1); stored as records, not a hardcoded list, so letting users change them later is a UI feature, not a redesign.
- **comparisons**: id, user_id, media_type (null for To-Watch, which crosses types), list_type, item_a, item_b, result (`a` | `b` | `tie`), source (`insertion` | `refinement` | `correction`), superseded_by (nullable), created_at

A Watched ranking is the ordered set of `user_items` for one (user, media_type). The To-Watch ranking is a single ordered list per user across all media types. The comparison log is append-only; a correction adds a new comparison and marks the conflicting older ones as superseded, so the log stays a full history for experiments.

**[DECIDE]** How `position` is stored: a dense integer (simple, but inserts renumber rows) or a fractional/sparse key (cheap inserts, occasional rebalancing). Either works at personal-list sizes.

## 6. Ranking Design

The design follows Beli's approach as described in public coverage and one third-party write-up: a rough bucket first, then a binary insertion into the existing ordered list by either-or choices. Beli does not publish its exact algorithm, so details below are our own.

### 6.1 Insertion (when an item is added)
1. **Bucket:** when the user first starts a media type, they choose how many buckets they want (default 3, allowed range 2 to 5) and can rename them. Default names **[PROPOSED]**: 2 buckets = liked it / didn't like it; 3 = liked it / it was fine / didn't like it; 4 = loved it / liked it / it was fine / didn't like it; 5 = loved it / liked it / it was fine / didn't like it / hated it. The choice is **locked in the MVP**. When adding an item, the user picks a bucket by gut feel, which narrows the range of positions the item can land in.
2. **Binary insertion:** compare the new item against the middle item of the bucket's range, then the middle of the remaining half, and so on. Each answer halves the range. About log2(n) questions for n items in the range (roughly 7 for 100).
3. **Answer options:** "A", "B", or "too tough." "Too tough" places the item tied with the compared one and stops narrowing. There is no skip option in the MVP (decided).
4. **Score:** derived from the final position (see 6.5).

To-Watch uses the same engine with the question "which do you want to watch first?"

### 6.2 Why insertion, and its weakness
Insertion is simple, intuitive, and keeps the questions per item small as the list grows. Its weakness is that each item is placed once: one wrong answer early in the search can leave an item far from where it belongs, and nothing repairs it. Sections 6.3 and 6.4 exist to fix that.

### 6.3 Refinement (occasional, on app open) **[NOT DECIDED YET]**
Occasionally show two items that are adjacent in the ranking and ask which is better. This lets the order self-correct over time without the user hunting for mistakes.

**Pair selection (MVP):** prefer adjacent pairs where one item was inserted recently or has few comparisons. Avoid repeating the same pair too soon. **[DECIDE]** the exact priority rule; tune with the benchmark.

### 6.4 Correction (user-initiated)
**MVP (decided):** the only correction is "Re-place this item" (flow 8): remove the item and run the normal placement again as if it were new. It reuses the placement code and needs no new algorithm. **Everything below is parked and not decided yet.**

If the user sees two items that look wrongly ordered, they select both and the app re-asks the comparison. Say item A currently sits below item B, and the user now says A is better:

1. Log the new comparison with `source = correction` and mark conflicting older comparisons superseded.
2. A must now sit above B, but how far above is unknown. Errors are usually small, so search outward from B with a **galloping search**: compare A against the items 1, 2, 4, 8, ... positions above B until A loses, then binary search inside that last interval. A small displacement costs only a few questions; a large one costs about twice log2(distance). The mirror case (A above B but the user says A is worse) gallops downward.
3. Cap the follow-up questions (for example 4 to 5). "Too tough" ties A with the item being compared and stops.
4. Show the result with an **undo**.

Only the corrected item moves; every other item keeps its relative order. Re-ranking a single item from the list works the same way, except the first question against an immediate neighbor tells the app which direction to gallop.

### 6.5 Score (display only) (decided: even spread)
The 0-10 range is split evenly across the user's buckets at setup (for example 3 buckets: about 6.7-10, 3.3-6.7, 0-3.3; 5 buckets: 2 points each). Beli's real mapping is not public. Items are spread evenly across their bucket's band by position. For an item at rank r (1 = best) in a bucket of n items, with band [lo, hi]:

score = hi - ((r - 0.5) / n) x (hi - lo)

This gives every item the middle of its own equal slice, so a lone item gets the middle of the band and no one reaches the exact extremes. Scores shift slightly as items are added; display one decimal. Tied items share the average of the scores of the slots they occupy. Users cannot edit scores directly.

### 6.6 Genre views **[NOT DECIDED YET]**
Genre lists are filters on the base ranking: the top rom-coms are the highest-ranked items tagged rom-com. They cost no extra questions and cannot disagree with the base order. An item with several genres appears under each. Genres come from provider metadata.

Optional tuning: prefer same-genre pairs during insertion so users are not asked to compare very different items. The overall order still needs some cross-genre comparisons to stay consistent, so this is a setting to test, not an assumption.

### 6.65 To-Watch list: place now or later (decided)
Items are often bookmarked in bulk, so adding to To-Watch must stay fast. When adding, the app asks "place it now or later?"

- **Place now:** the usual either-or questions ("which do you want to watch first?") put the item in the ranked To-Watch list. The top of the list is the next pick.
- **Later:** the item goes into a separate "to place" queue. It is not in the ranked list. The user can open the queue at any time and place items one at a time.
- A small count badge on the queue (for example "12 to place") keeps it from becoming a graveyard. An occasional gentle reminder when the app opens is an option **[DECIDE]**.
- An item can be started straight from the queue without being placed (see flow 5).
- The ranked To-Watch list uses no buckets and shows no scores: it is just an ordered list where the top is the next pick (decided).
- **One combined list across all media types** (decided). Placement questions compare the new item against items of any type, because "what do I want to do next?" is a question about desire and mood, which makes sense across movies, TV, anime, and games. Type filters (and genre filters) are views of this one list and need no extra questions. Watched stays ranked separately per media type, since comparing quality across types is awkward.
- Tradeoffs: answers will depend more on mood and on commitment (a 2-hour movie versus a 60-hour game), so this ranking is noisier than Watched. A later filter such as "I have 2 hours tonight" would need runtime data from the providers.

### 6.7 Extensibility: provider adapters
The ranking engine only sees generic items and comparisons; it knows nothing about movies or games. All type-specific code lives behind one interface:

- `search(query) -> list[ItemDTO]` (ItemDTO includes title, year, image, genres)
- `get(external_id) -> ItemDTO`

Each source (TMDB, AniList, IGDB) implements it and registers under a `media_type`. Adding a new type means writing one adapter, registering it, and adding a UI tab. A shared test suite runs against every adapter (with recorded API responses).

### 6.8 Later: Bayesian / uncertainty-aware ranking
Use Bradley-Terry or TrueSkill so each item has a score and a confidence, and ask about the items we are least sure of. Only if the benchmark shows insertion plus refinement is not enough.

## 7. Evaluation Plan

Goal: show how accurately the ranking recovers a user's true preferences, and how many questions it takes.

1. Generate synthetic users with a hidden "true" ordering of N items.
2. Simulate answers with noise. **[DECIDE]** the noise model: a flat error rate (for example 5-10%) is the simplest, but real mistakes are likelier between items that are close in true rank, so also test a distance-based model. State the assumption in the README.
3. Compare strategies:
   - (a) binary insertion only,
   - (b) insertion plus refinement questions,
   - (c) insertion plus refinement plus corrections (simulated users who fix visible errors).
4. Measure Kendall tau or Spearman against the true ordering, questions per insertion, and total questions.

Results go in the README. Use only numbers actually measured here; none are claimed yet.

## 8. Testing Plan

- Unit tests: binary insertion (including bucket ranges and ties), correction range logic, score mapping, genre filtering.
- API tests: add item, answer an insertion question, submit a correction, get a ranking.
- Adapter tests: the shared suite against recorded API responses.
- CI: GitHub Actions runs pytest on every pull request; merging requires a passing build.

## 9. Milestones

1. **M0, Setup:** repo, design doc, skeleton, Docker Compose with Postgres, CI.
2. **M1, Items:** provider interface, TMDB adapter (movies, TV), add to lists, accounts.
3. **M2, Ranking engine:** bucketed binary insertion, ties, score, tests. *(Write or re-derive this yourself.)*
4. **M3, Comparison UI and rankings view:** insertion flow, Watched scores, the To-Watch list with its place-now-or-later flow and a media type filter, and the In Progress shelf with its Start and Finish actions.
5. **M4, Refinement and correction:** occasional refinement questions and the two-item correction flow.
6. **M5, More types:** AniList adapter (anime), then IGDB adapter (games). Each should need only a new adapter and a UI tab; if it needs more, fix the abstraction.
7. **M6, Benchmark and deploy:** simulation results, README, live link.
8. **M7, Friend recommendations (after the core works):** friend connections, an inbox, and accept/dismiss that sends an accepted recommendation to To-Watch tagged with who recommended it.

Each milestone ends with an explain-back check (whiteboard, no code open) before the next begins.

## 10. Workflow Rules

- One ticket per branch, one branch per pull request. Nothing goes straight to main.
- Builder agent implements, reviewer agent critiques, and the author reads every diff.
- A change merges only when CI passes and the author can explain it from memory.
- Keep a decisions log (`docs/decisions.md`): what changed and why.

## 10.5 Launch and Compliance

The app is deployed publicly on the web for personal use and friends. It is free, with no ads or payments.

- **Attribution:** show TMDB (and later AniList/IGDB) attribution and logos in an About/Credits page. Re-read each source's current terms before launch.
- **Commercial terms:** TMDB's free API is for non-commercial use, and IGDB requires a partnership for commercial use. Staying free and non-monetized keeps the app on the non-commercial side. If that ever changes, contact each provider first.
- **Privacy:** a short privacy policy and terms page; collect only what is needed (email, comparisons).
- **Account deletion:** a way to delete your account and data.
- **Security:** provider-managed auth, rate limiting, and API keys kept on the backend only, never in client code.
- **Operations:** error monitoring, regular Postgres backups, and a check that the live link is up.

## 11. Risks

- **Scope creep:** friends, imports, cross-media ranking, and genre-only rankings are deferred on purpose.
- **Noisy answers in insertion:** one wrong early answer misplaces an item. Mitigated by refinement, corrections, and the benchmark.
- **Time:** if short, cut features, not the review habits. First cut: games (IGDB credentials and quirks), then anime.
- **Leaky abstraction:** type-specific details creeping into the ranking engine. Mitigated by the adapter interface and the rule in M5.
- **Forced odd comparisons:** users dislike being asked to compare very different items. Mitigated by buckets, the "too tough" option, and optional same-genre pairing.

## 12. Open Questions

- Which deploy target?
- Which auth provider?
- How is `position` stored (dense integer or fractional key)?
- Does To-Watch use buckets, or one flat priority order?
- Score mapping details (band sizes, tie handling, "provisional" threshold)?
- Refinement pair priority rule?
- Noise model for the benchmark?
- Should changing an item's bucket re-run insertion automatically?