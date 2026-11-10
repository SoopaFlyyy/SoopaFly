# Sorare NBA Engine — External Research & Integration Plan

Research date: 2026-11-10

## Reference repository

https://github.com/script-money/sorare-nba-lineup-tool

The reference repository is public. Its `pyproject.toml` declares an MIT license, although the inspected repository tree does not list a separate LICENSE file. Its README says it was working for the 2023-24 season. It is a legacy implementation: use its documented algorithms and API patterns as research, not as a drop-in current rules engine.

## Useful techniques verified in source

1. **Distribution-based score estimates**
   - `compute.py` models projected player performance using `NormalDist`, estimates dispersion from recent fixture scores, uses a Student-t fit for robustness, and smooths recent results with EWMA.
   - Keep both expected score and uncertainty. Do not report only a single deterministic projection.
   - Modernize the model with explicit sample-size minimums, zero-minute/DNP filtering, robust variance estimation, and calibration against actual outcomes. Avoid trusting the legacy hardcoded assumptions without validation.

2. **Rotation / replacement context**
   - `position_analysis` maps starter, second and third options for a team role.
   - `query_injure_and_match.py` ingests injury and schedule data; it also marks back-to-back games.
   - Modernize as lineup-based role data and teammate-on/off splits: minutes, usage, rebounds, assists, stocks, turnovers, and Sorare score with/without unavailable teammates. Track co-absences and sample size to avoid false injury boosts.

3. **Opponent and schedule effects**
   - The legacy predictor adjusts estimates for team/opponent offense and defense ranks, home/away, B2B and game count.
   - Keep schedule context as small, evidence-backed modifiers; do not let stale overall team ranks dominate positional matchup, pace, likely defender, or role evidence.

4. **Contest eligibility from Sorare**
   - `query/NBAOpenFixtureQuery.graphql` retrieves open fixture, leaderboard metadata, requirements, allowed rarities and existing lineups.
   - `query/NBAMyCardsSuggestionsQuery.graphql` calls `nbaLeaderboard(slug: ...).myComposeLineupCards` with `includeUsed: false` and pagination.
   - Prefer Sorare's current server-provided eligibility/suggestion result as a validation layer, then optimize locally. Never infer eligibility solely from card labels.

5. **Card-aware lineup operations**
   - The repo retains card slugs/IDs and has create/update and delete lineup mutations.
   - Any write integration must be explicitly gated, present a review screen, validate contest rules and card uniqueness, and require a user confirmation. Never auto-submit.

6. **Player and card data acquisition**
   - Separate jobs query all cards, all players, current player positions, injuries and match schedules.
   - Price history and overperformance data are auxiliary signals; keep them out of the core score projection unless validated.

## Legacy limitations to fix rather than copy

- README targets 2023-24 season.
- Tournament rules and target values are hardcoded in `config/config.py`; current season formats must come from fresh Sorare contest metadata and verified current rules.
- Injury probabilities are static fallback values (e.g. Probable 0.75, Questionable 0.5, Doubtful 0.25). These are not reliable calibrated availability probabilities. Keep categorical status and use source-specific evidence; do not turn tags into unsupported numeric certainty.
- Injury ingestion uses CBS Sports scraping and old official report URL/PDF handling. Replace with timestamped official NBA report plus independent ESPN, Underdog NBA, RotoWire cross-checks.
- Historical data files are cached in local JSON/pickle and are not a complete modern event store.
- The old score formula has turnovers at -2 in its custom-ratio path, whereas this project must enforce the applicable Sorare NBA scoring rules and the user's explicitly required turnover penalty of -3 when calculating the user's relevant contests.
- Its roster-role map is manually refreshed weekly. Modern replacement must use recent starters, minutes hierarchy and role changes.

## Integration plan for our engine

### P0 — Preserve / validate existing foundations
- Keep the existing Sorare OAuth implementation and session; do not create a second Sorare login flow.
- Keep owned card identity at card level: asset ID / card slug, rarity, season, serial, XP, grade, bonus, In-Season eligibility, used/locked state.
- Add provenance and timestamps to each external data record.

### P1 — Injury and rotation impact
- Normalize official NBA, ESPN, Underdog and RotoWire statuses without silently overwriting conflicts.
- Track every source's retrieved/published timestamp.
- Mark OUT as ineligible immediately; retain Q/Doubtful/GTD as scenarios with explicit risk.
- Compute teammate on/off samples with minutes, usage, rebound/assist rates, defensive stats and Sorare scores; exclude/cohort games with other key players absent; expose n and confidence.
- Add opponent-out analysis separately: assess likely defender, rim protection, rebounding, pace and replacement matchup; no automatic boost.

### P2 — Distribution-based projection
- Produce score mean, median, standard deviation, conservative percentile, ceiling percentile and minutes distribution.
- Incorporate last 3/5/10 games, season baseline, role trend, starting/bench role, expected minutes, matchup, pace, injury impact, B2B/rest and blowout risk.
- Apply exact card bonus to the card projection, not to player baseline; preserve separate base and boosted scores.
- Calibrate weekly with prediction-vs-actual residuals and reliability buckets.

### P3 — Shared inventory optimization
- Solve all relevant decks jointly, using unique card IDs, not player names.
- Two Champion and four Contender Limited In-Season decks: exactly four In-Season plus one Classic per deck, with current verified Star caps and eligibility.
- Enforce no card duplication across simultaneously submitted lineups, used/locked exclusions, and per-competition rarity.
- Optimize objective per contest (expected score, probability of threshold/reward, risk-adjusted utility); return impossible deck explanations where constraints cannot be met.

### P4 — Hot Streak target optimizer
- Model the total score distribution with actual per-card bonuses.
- Optimize probability of clearing the target threshold, not only sum of means.
- Run replacement what-if scenarios and sensitivity to uncertain statuses / minutes.
- Use historical lineup outcomes to calibrate target probabilities.

## Safety and rollout
- Implement on a separate Git branch and Vercel preview.
- Add unit tests for status normalization, bonus application, eligibility constraints, card uniqueness, Star cap, and distribution/probability calculations.
- Keep production unchanged until preview and test verification succeeds.
- Never expose Sorare secrets, OAuth client secrets, tokens or session cookies in Git.
