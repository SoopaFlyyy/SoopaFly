# Sorare NBA Engine — Technical Blueprint

This document converts the external research into an implementation contract for the existing Sorare NBA cloud engine.

## Current deployed baseline (inspected 2026-11-10)

Vercel project: `sorare-nba-engine`
Production domain: `sorare-nba-engine.vercel.app`
Deployed source files observed:
- `src/src/api/handler.js`
- `src/src/api/injuries.js`
- `src/src/index.html`
- `src/src/package.json`
- `src/src/vercel.json`

The deployed `handler.js` already contains:
- Sorare OAuth authorize/token endpoints and session cookie/state names.
- Existing owned NBA cards GraphQL query via `cardsWhere`, paginated 50 at a time.
- Card fields including rarity, grade, XP, power, In-Season eligibility, card set/edition, positions, team, suspensions, task appearances, opened/live SO5 lineups and used-in state.
- Player fields including active injuries, L10/L5/season average, minutes L10/L5, points/rebounds/assists/turnovers L10 and next-game context.

The deployed `injuries.js` currently scrapes PropVig HTML. This is a single-source weakness.

Current cron in `vercel.json`: `/api/injuries` at `0 22 * * *` UTC (05:00 Asia/Ho_Chi_Minh). Add intra-slate refreshes and near-lock refreshes, subject to provider reliability and plan limits.

## Required service boundaries

Keep the existing OAuth callback and cookie/session implementation intact. Do not create a second login or credentials flow.

Recommended modules:
- `src/src/lib/injury-sources.js`: source adapters + timestamps + raw normalized status.
- `src/src/lib/injury-normalizer.js`: player/team mapping, canonical status, source conflict rules.
- `src/src/lib/injury-impact.js`: teammate-on/off and opponent absence analysis.
- `src/src/lib/projection.js`: minutes and score distributions.
- `src/src/lib/sorare-rules.js`: explicit contest configurations backed by current official metadata/rules.
- `src/src/lib/lineup-optimizer.js`: card-level single and joint deck allocation.
- `src/src/api/analysis.js`: read-only analysis API.
- `src/src/api/lineups.js`: read-only optimizer output and validation. Sorare write operations remain disabled until explicit user confirmation.
- `src/src/lib/store.js`: persisted snapshots / cache, if a supported database is configured.

Do not deploy empty placeholder files or change runtime routes before tests exist.

## Canonical data contracts

### Injury source record

```json
{
  "player_slug": "player-slug",
  "player_name": "Player Name",
  "team": "Team",
  "source": "NBA_OFFICIAL",
  "source_published_at": "2026-11-10T00:00:00Z",
  "retrieved_at": "2026-11-10T00:05:00Z",
  "status_raw": "Questionable",
  "status_normalized": "QUESTIONABLE",
  "reason": "Right ankle sprain",
  "game_date": "2026-11-10",
  "source_url": "https://..."
}
```

Keep one record per player/source/report. Never destroy source disagreement by overwriting the earlier report.

Canonical statuses:
- OUT
- DOUBTFUL
- QUESTIONABLE
- PROBABLE
- AVAILABLE
- GTD (only when source literally uses GTD; map into QUESTIONABLE risk family for selection but preserve raw tag)
- UNKNOWN
- NOT_LISTED (not a confirmation of AVAILABLE)

Source precedence:
1. Latest official NBA report is the authoritative league status.
2. ESPN, Underdog NBA and RotoWire are independent cross-checks and useful for update/news context.
3. Conflicts remain visible with timestamps; do not silently promote a player from questionable to available based on a secondary source.
4. A stale report cannot overwrite a newer report.

### Projection record

```json
{
  "player_slug": "player-slug",
  "game_id": "provider-game-id",
  "as_of": "2026-11-10T00:05:00Z",
  "expected_minutes": 31.5,
  "minutes_sd": 4.8,
  "base_score_mean": 47.2,
  "base_score_sd": 13.1,
  "p10": 29.0,
  "p50": 46.0,
  "p90": 65.0,
  "sample": {
    "l3": 3,
    "l5": 5,
    "l10": 10,
    "season": 18,
    "with_absent_teammate": 4
  },
  "risk_flags": ["ROLE_VOLATILITY"],
  "evidence": []
}
```

A missing player from a report is NOT AVAILABLE. An uncertain player must be modelled with explicit scenarios (plays / sits) rather than a single fixed probability until availability probabilities are calibrated.

### Card record

Use a stable Sorare card identity:
- `card_id` or `asset_id` (preferred unique key)
- card slug
- player slug
- rarity
- season / serial
- `in_season_eligible`
- bonus and bonus components
- XP / grade / power
- used/locked flags and reason
- open/live lineup IDs
- selection eligibility and eligibility source/timestamp

The player projection is shared across copies; the card projection applies that card's exact bonus. Do not collapse multiple copies by player name.

## Injury impact method

For each material OUT / uncertain player:
1. Find current teammates and same-position / role alternatives.
2. Build a game-log cohort for the target player with the star absent and present.
3. Exclude or separately label games where additional high-impact teammates were absent.
4. Report n, minutes, usage, points, rebounds, assists, steals, blocks, turnovers, Sorare score and role/starting rate.
5. Shrink small-sample deltas toward season/role baseline; never publish raw small-sample difference as the projection.
6. Separate:
   - teammate usage / minutes impact,
   - opponent defensive matchup impact,
   - pace / game script,
   - replacement defender or rim protection impact.
7. Provide no positive adjustment unless there is a basketball mechanism and supporting sample.

## Score model

The reference project's NormalDist / EWMA / robust distribution concept is retained, but upgraded.

Projection inputs:
- L3, L5, L10, season and same-role baseline.
- Recent expected minutes and minutes distribution.
- Starter/bench + role trend.
- Usage and on-ball role.
- Rebound / assist opportunities.
- Stocks and turnover profile.
- Team pace, opponent pace and defensive profile.
- Positional matchup / likely defender / rim protection.
- Rest and back-to-back.
- Blowout and foul risk.
- Injury on/off deltas with sample-size shrinkage.

Outputs must include mean, uncertainty and percentiles. Calibrate on historical Sorare results, not generic box score scoring alone.

Apply exact Sorare scoring rules in one versioned rule module. Verified Sorare NBA help currently specifies turnovers at -2. The user's earlier -3 preference conflicts with Sorare's published scoring matrix; do not use -3 in official Sorare score projections. Keep the scoring matrix versioned and re-check official rules when Sorare publishes a change.

For any card:
`boosted_score = base_score * (1 + exact_card_bonus)`
Keep base and boosted projections separate. Confirm Sorare bonus composition against actual card metadata before applying.

## Contest and lineup optimizer

First read current Sorare contest metadata and requirements. Use the modern equivalents of:
- `nbaOpenFixture`
- leaderboard requirements / rarity eligibility
- Sorare server suggestion endpoint `myComposeLineupCards` (including pagination and `includeUsed: false`)

Treat Sorare's eligibility as a validation source. Local optimizer still enforces explicit constraints.

NBA 27 (2026-27) strategy baseline, pending live Sorare API validation:
- Five-card lineup; no position restrictions.
- Star budget replaces points cap and MVP: Contender 11 Stars, Champion 16 Stars; player Star rating 1–5 in half-star increments, based on L10 and refreshed at Game Week deadline.
- Do not hardcode the old 120/140 points caps or MVP logic into NBA 27 contests.
- In-Season Pro/leaderboard lineup rules still require at least four In-Season cards and allow at most one Classic card where the relevant contest specifies that format; verify current requirements from Sorare metadata for each leaderboard.
- Classic competitions permit past-season cards; do not apply In-Season card restrictions there unless the contest metadata says otherwise.
- The guide states Daily Pickup is replaced by Arcade in NBA 27. Retire Daily Pickup-specific assumptions unless the live Sorare API exposes a current equivalent.
- For Game Week contests, only each player's highest-scoring game within the GW counts. Model each player's GW score as max(score across scheduled games), not sum or average. More games create more opportunities but require minutes/availability risk per game.
- Current game-week split described by the guide: Monday–Thursday and Friday–Sunday; verify actual fixture start/end timestamps from Sorare.
- OUT cards excluded from safe lineups; Q/D/GTD retained as scenario risks.
- Use current Sorare server eligibility and contest metadata as the final authority; treat this strategy guide as secondary reference.

The previous project plan's fixed six-deck target (2 Champion + 4 Contender) is an optimizer target only if those are the competitions currently available and the user still wants that allocation. It is not a Sorare rules requirement. Solve requested lineups jointly with card-ID uniqueness where the competitions overlap in time. Report infeasible decks and closest legal alternatives.

## Hot Streak optimizer

For 5-card lineup:
- Apply each card's exact bonus separately.
- Calculate distribution of total score, including covariance for correlated same-game / same-team players where data supports it.
- Calculate P(total >= target) for user-specified target (e.g. 270).
- Include downside (P10), median, mean, ceiling (P90), and injury sensitivity.
- Search replacements and show marginal delta in probability of clearing target, not merely delta to mean.

Do not invent a probability if score distributions have not been calibrated. Label it experimental until calibration data is adequate.

## Tests required before feature activation

Unit tests:
1. Latest source timestamp wins within same source.
2. Secondary sources never overwrite official report.
3. Missing from NBA report maps to NOT_LISTED, not AVAILABLE.
4. OUT player removed from safe candidates.
5. Q/GTD retained as uncertain scenario with explicit flag.
6. Card bonus applied exactly once.
7. Multiple card copies of same player compete by card-level projected value.
8. A card ID cannot be reused across six decks.
9. Each deck meets 4 In-Season + 1 Classic.
10. Each deck meets competition rarity and current star cap.
11. Locked / used cards cannot be selected.
12. Distribution aggregate uses correct variance/covariance.
13. Probability threshold calculation matches known synthetic cases.
14. Turnovers scored -3.
15. Missing sample data falls back to baseline without fabricating injury impact.

## Rollout
- All new work starts on `sorare-engine-integration` branch.
- Build preview only, never point a new Vercel deployment at production alias.
- Run tests and inspect preview APIs/UI.
- Only after validation, create a PR and review before merge.
- Never push Sorare tokens, OAuth client secrets, session cookies, or raw account data into Git.


## NBA 27 strategy-guide update (2026-11-10)

Reference: https://sirius-sports.github.io/sorare-origin/en/nba-strategy

This is an unofficial guide, so it informs strategy but never overrides Sorare's live contest metadata or official help.

Key implications for the model:
1. **Star-value optimization:** calculate value as projected GW score relative to Star cost, and identify players whose current role/minutes/usage exceed what their stale L10-derived Star rating implies. Track the Star rating snapshot and the deadline at which it refreshes.
2. **Five-card no-position roster:** optimizer slots are five generic slots unless contest metadata imposes another restriction.
3. **Champion / Contender budgets:** use 16 Stars / 11 Stars, respectively, with half-star player ratings. Treat as 2026-27 guide values; verify against the live leaderboard.
4. **Best game in the GW:** a player's GW contribution is the maximum valid game score across the GW, not the sum. Model game-by-game injury, minutes, matchup and ceiling, then calculate the distribution of the maximum.
5. **Volatility matters:** include ceiling / upper-percentile outcomes for top finishes, but retain downside and probability of a DNP. Optimize objective by contest type.
6. **Injury-L10 lag:** identify backups and 6th/7th men whose projected role changes before the Star rating catches up. This directly supports the existing user priority to value rotation players.
7. **Bonus stacking:** card-specific rarity, XP, season and collection bonuses must be read from actual Sorare card/contest data and applied once. Do not hardcode guide bonus values where live card metadata or rules can differ.
8. **Injury diligence:** the guide calls out NBA injury reports, ESPN and RotoWire. Continue the existing official NBA + ESPN + Underdog + RotoWire cross-check pipeline.
9. **Market signals:** price / XP / transfer cost / season status belong in a separate card acquisition-value model, not the pure basketball projection.
10. **DNP/NG:** both produce zero; scheduled games and expected participation are first-class risk inputs.

### Scoring rule conflict resolved
The guide and Sorare's official help article both list turnovers as -2, not -3. Official Sorare help is the source of truth for scoring: https://help.sorare.com/hc/en-us/articles/7245592557725-How-are-player-scores-calculated-in-Sorare-NBA. The previous -3 note in this blueprint was incorrect for official Sorare scoring and is superseded. Keep the score matrix versioned and sourced.

### Strategy guide limitations
- Unofficial and explicitly disclaims affiliation/guaranteed accuracy.
- Some bonus descriptions may be simplified or season-specific; use current Sorare card/leaderboard fields.
- Do not assume every competition has identical entry limits, card eligibility, reward rules or lock times. Read live competition requirements.
