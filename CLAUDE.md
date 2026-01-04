# Supa Bet - Sports Betting Agent System

## What This Is

A hierarchical agent system for sports betting predictions using Claude Code skills and agents. Follows the LATM (Large Language Models as Tool Makers) pattern where single-purpose agents prevent hallucinations by isolating tasks and verifying each other.

## Quick Start

```bash
# Automated: pg_cron refreshes odds every 30 min on gameday, every 2hr for tomorrow's games

# Manual override: Force refresh all sports
/refresh-games

# Analyze games and make picks
/predict

# After games complete, check results
/settle-bets

# Find games 3-7 days out (usually automatic via cron)
/discover-games
```

## Architecture

```
═══════════════════════════════════════════════════════════════════
                    AUTOMATED DATA PIPELINE (pg_cron)
═══════════════════════════════════════════════════════════════════

                     ┌─────────────────────────────────────────┐
                     │           pg_cron SCHEDULER             │
                     │  ├── refresh-opening (daily 9am UTC)    │
                     │  ├── refresh-72hr (every 6hr)           │
                     │  ├── refresh-24hr (every 2hr)           │
                     │  ├── refresh-gameday (every 30min)      │
                     │  └── settle-games (8pm/10pm/midnight)   │
                     └─────────────────────────────────────────┘
                                        │
                                        ▼ (pg_net HTTP)
                     ┌─────────────────────────────────────────┐
                     │     EDGE FUNCTION: refresh-odds (v12)   │
                     │  1. Fetch ESPN scoreboard (event IDs)   │
                     │  2. Fetch ESPN odds API (signed spread) │
                     │  3. Fetch ESPN injuries API (JSON)      │
                     │  4. Fetch DailyFaceoff goalies (NHL)    │
                     │  5. Upsert all data atomically          │
                     └─────────────────────────────────────────┘
                                        │
                                        ▼
                     ┌─────────────────────────────────────────┐
                     │              SUPABASE                   │
                     │  ├── games (+ espn_event_id)            │
                     │  ├── game_odds (current lines)          │
                     │  ├── odds_history (line movement)       │
                     │  ├── game_injuries (linked to games)    │
                     │  ├── injury_history (status tracking)   │
                     │  └── game_goalies (NHL starters)        │
                     └─────────────────────────────────────────┘

═══════════════════════════════════════════════════════════════════
                    MANUAL OVERRIDE: /refresh-games
═══════════════════════════════════════════════════════════════════

/refresh-games → refresh-orchestrator → Edge Function OR ESPN JSON APIs
                                        (force refresh, bypass schedule)

═══════════════════════════════════════════════════════════════════
                    PREDICTION (Split Agents)
═══════════════════════════════════════════════════════════════════

/predict (Lightweight Orchestrator)
    │
    ├── Query TODAY's games from game_analysis_view
    │
    └── FOR EACH GAME (fresh context):
            │
            ├── Step 1: Task(haiku) → game-researcher
            │   ├── WebFetch: Action Network public betting
            │   ├── WebSearch: Injuries
            │   ├── WebSearch: Lineups/scratches
            │   └── Returns: Research JSON (~3-4k tokens)
            │
            └── Step 2: Task(opus) → game-predictor
                ├── Receives: Game data + Research JSON
                ├── Rates: spread/ML/total confidence (0-100)
                ├── Decides: BET or PASS
                ├── Writes: prediction_log + placed_bets
                └── Returns: Short summary (~8-10k tokens)

            [CONTEXTS DISCARDED - next game fresh]

Key: Haiku researches (cheap), Opus decides (quality). ~50% token reduction.

═══════════════════════════════════════════════════════════════════
                    PHASE 3: SETTLEMENT
═══════════════════════════════════════════════════════════════════

/settle-bets → ESPN scoreboard → Match scores → Update placed_bets
                                               (result, final_score)
                                                      │
                                           cleanup_old_games()
```

## Supabase Setup

**Project**: `bet`
**Project ID**: `htnhszioyydsjzkqeowx`
**Region**: us-west-2

### Connection

The Supabase MCP is already connected. Use:
- `mcp__supabase__execute_sql` for queries
- `mcp__supabase__list_tables` to see schema

### Tables

| Table | Key Columns | Purpose |
|-------|-------------|---------|
| `games` | sport, game_date, away_team, home_team, espn_event_id, status | Core game data |
| `books` | name, short_name, priority | Sportsbook reference (DK, FD, MGM, CZR) |
| `game_odds` | game_id, book_id, spread, total, away_ml, home_ml | Current odds per book |
| `odds_history` | game_id, spread, total, open_spread, open_total, hours_before_game | Line movement snapshots |
| `game_injuries` | game_id, player_name, team, side, status | Player injury reports (linked to games) |
| `injury_history` | sport, team, player_name, status, previous_status, first_reported | Tracks when injuries announced + status changes |
| `game_goalies` | game_id, away_goalie_name, home_goalie_name, *_confirmed, *_gaa, *_sv_pct | NHL starting goalies from DailyFaceoff |
| `placed_bets` | sport, game_date, bet_type, selection, odds, result, game_context | Immutable bet records |
| `prediction_log` | game_id, decision (BET/PASS), public_pct, sharp_detected, reasoning, result | All decisions for ML |
| `odds_history_archive` | game_snapshot (JSONB), recorded_at, spread, open_spread | Preserved odds after game cleanup |
| `injury_history_archive` | game_snapshot (JSONB), recorded_at, player_name, status | Preserved injuries after game cleanup |

> **Note**: Use `mcp__supabase__list_tables` for full schema. Agents should query `game_analysis_view` which pre-aggregates all data.


### Views

#### `game_analysis_view` - Aggregated game data for agents

Pre-aggregates all game data into a single row per game with nested JSON:

```sql
SELECT * FROM game_analysis_view WHERE sport = 'NBA';
```

Returns columns:
- `game_id`, `sport`, `game_date`, `game_time`, `away_team`, `home_team`, `status`, `espn_event_id`
- `odds` - Best available odds (JSON object)
- `all_odds` - All book odds (JSON array)
- `injuries` - Grouped by away/home (JSON object)
- `line_movement` - Enhanced tracking (JSON object):
  - `opening_spread`, `current_spread`, `spread_change`, `spread_direction`
  - `opening_total`, `current_total`, `total_change`
  - `hours_until_game`

### Functions

#### `cleanup_old_games(days_old INTEGER DEFAULT 7)`

Archives odds and injury history to `*_archive` tables, then removes games older than X days with terminal status. Preserves historical data for ML/backtesting.

```sql
SELECT * FROM cleanup_old_games(7);  -- Returns sport, deleted_count, archived_odds, archived_injuries
```

#### `mark_stale_games()`

Marks games from past dates still marked as 'scheduled' as 'stale'.

```sql
SELECT mark_stale_games();  -- Returns count of games marked stale
```

### Key Queries

```sql
-- Primary query for agents (use this!)
SELECT * FROM game_analysis_view WHERE game_date >= CURRENT_DATE;

-- Win rate
SELECT result, COUNT(*) FROM placed_bets WHERE result != 'PENDING' GROUP BY result;
```

## Edge Functions

All Edge Functions can be triggered manually via HTTP POST:

```bash
# Base URL
https://htnhszioyydsjzkqeowx.supabase.co/functions/v1/{function-name}
```

### `refresh-odds` - Automated Odds + Injuries + Goalies (v12)

**Atomic data capture**: Fetches odds, injuries, and goalies in a single function call so all data has the same timestamp. This enables line movement correlation with injury announcements.

**Endpoint**: `https://htnhszioyydsjzkqeowx.supabase.co/functions/v1/refresh-odds`

**Parameters**:
```json
{
  "tier": "opening" | "72hr" | "24hr" | "gameday",
  "sport": "NBA" | "NHL" | "NFL" | "NCAAB",  // optional
  "force": true  // skip schedule checks
}
```

**Manual trigger**:
```bash
curl -X POST https://htnhszioyydsjzkqeowx.supabase.co/functions/v1/refresh-odds \
  -H "Content-Type: application/json" \
  -d '{"force": true}'
```

**What it does**:
1. **Phase 1 - Odds**: Fetches ESPN scoreboard → ESPN odds API → upserts `game_odds` + `odds_history`
2. **Phase 2 - Injuries**: Fetches ESPN injuries API → upserts `game_injuries` + `injury_history`
3. **Phase 3 - Goalies** (NHL only): Fetches DailyFaceoff `__NEXT_DATA__` → upserts `game_goalies`

**Sample response**:
```json
{
  "summary": {
    "total": 17,
    "updated": 17,
    "injuries": { "fetched": 61, "updated": 209, "newReports": 61 },
    "goalies": { "fetched": 4, "updated": 4 }
  }
}
```

### `refresh-injuries` - DEPRECATED (Replaced)

~~Old HTML scraper - broken when ESPN changed their page structure.~~

**Status**: Replaced by integrated injury fetching in `refresh-odds` v11. The Edge Function still exists but is not scheduled.

**New approach**: ESPN has a JSON injuries API that works reliably:
```
https://site.api.espn.com/apis/site/v2/sports/{sport}/{league}/injuries
```

### `settle-games` - Game Settlement

Fetches final scores from ESPN and settles completed games + bets.

**Endpoint**: `https://htnhszioyydsjzkqeowx.supabase.co/functions/v1/settle-games`

**Parameters**:
```json
{
  "date": "2025-12-29",  // optional, defaults to yesterday
  "sport": "NBA"         // optional, defaults to all sports
}
```

**Manual trigger**:
```bash
# Settle yesterday's games (default)
curl -X POST https://htnhszioyydsjzkqeowx.supabase.co/functions/v1/settle-games

# Settle specific date
curl -X POST https://htnhszioyydsjzkqeowx.supabase.co/functions/v1/settle-games \
  -H "Content-Type: application/json" \
  -d '{"date": "2025-12-28"}'
```

**What it does**:
1. Fetches ESPN scoreboard for completed games
2. Updates `games` table: `status = 'final'`, `final_score`
3. Settles `placed_bets`: `result` (WIN/LOSS/PUSH), `final_score`, `settled_at`
4. Parses bet selection to determine outcome (spread, total, moneyline)

## Cron Jobs (pg_cron)

Automated scheduled refreshes via Supabase pg_cron:

| Job | Schedule | Target Function | Purpose |
|-----|----------|-----------------|---------|
| `refresh-opening-lines` | Daily 9am UTC | `refresh-odds` | Discover games 3-7 days out, capture opening lines |
| `refresh-72hr` | Every 6 hours | `refresh-odds` | Games 2-3 days out |
| `refresh-24hr` | Every 2 hours | `refresh-odds` | Games tomorrow |
| `refresh-gameday` | Every 30 min | `refresh-odds` | Today's games |
| `settle-games` | 6am/8am/10am UTC | `settle-games` | Settlement checks |

**Check cron status**:
```sql
SELECT * FROM cron_job_status;
SELECT * FROM cron.job_run_details ORDER BY start_time DESC LIMIT 10;
```

## ESPN JSON APIs

Structured JSON endpoints (no HTML scraping needed):

| API | URL Pattern |
|-----|-------------|
| Scoreboard | `https://site.api.espn.com/apis/site/v2/sports/{sport}/{league}/scoreboard?dates=YYYYMMDD` |
| Odds | `https://sports.core.api.espn.com/v2/sports/{sport}/leagues/{league}/events/{EVENT_ID}/competitions/{EVENT_ID}/odds` |
| Injuries | `https://site.api.espn.com/apis/site/v2/sports/{sport}/{league}/injuries` |

**Sport paths**:
- NBA: `basketball/nba`
- NHL: `hockey/nhl`
- NFL: `football/nfl`
- NCAAB: `basketball/mens-college-basketball` (injuries returns empty)

## DailyFaceoff API

NHL goalie data extracted from `__NEXT_DATA__` JSON embedded in HTML:

| Data | URL |
|------|-----|
| Starting Goalies | `https://www.dailyfaceoff.com/starting-goalies/` |
| Line Combinations | `https://www.dailyfaceoff.com/teams/{team-slug}/line-combinations` |

**Goalie confirmation levels**: `Confirmed`, `Probable`, `Unconfirmed`

## Skills Reference

| Skill | Command | Purpose |
|-------|---------|---------|
| `refresh-games` | `/refresh-games` | Manual override to refresh games (calls Edge Function) |
| `discover-games` | `/discover-games` | Find games 3-7 days out for opening lines |
| `predict` | `/predict` | Lightweight orchestrator - spawns fresh opus per game |
| `settle-bets` | `/settle-bets` | Check results and grade pending bets |

## Agents Reference

| Agent | Model | Purpose |
|-------|-------|---------|
| `refresh-orchestrator` | - | Calls Edge Function or ESPN JSON APIs, updates Supabase |
| `game-researcher` | Haiku | WebFetch Action Network (bet%) + SportsBettingDime (money%) + WebSearch injuries/lineups |
| `game-predictor` | Opus | Receives research JSON, rates confidence (0-100) per bet type, makes BET/PASS decision |

## Key Design Decisions

| Decision | Choice | Why |
|----------|--------|-----|
| Split agents | Haiku researches, Opus decides | Cost efficiency + quality decisions |
| Inline research | WebFetch + WebSearch per game | Always fresh data, no stale cron |
| Dual betting sources | Action Network (bet%) + SportsBettingDime (money%) | Sharp detection via divergence |
| Fresh context per game | Task agents discarded after each game | No context bleed between games |
| Log ALL decisions | prediction_log table | Learning requires knowing why we passed, not just bet |
| Full autonomy | No rigid rules | Agent decides when to bet based on its own analysis |
| Re-prediction allowed | No UNIQUE on game_id | Can re-analyze as lines move |

## Decision Philosophy

**Full autonomy - no guardrails.**

The agent receives game data, does research, and makes its own BET/PASS decision. No prescribed thresholds, no forced PASS conditions, no "safe" defaults.

- Missing public data? Analyze anyway with available signals.
- Signals conflict? Make a judgment call.
- Uncertain? Bet and learn from the outcome.

All decisions logged to `prediction_log` for learning. The agent develops intuition through experience, not rules.

## Data Sources

| Source | Used For | Frequency | URL |
|--------|----------|-----------|-----|
| ESPN Scoreboard API | Event IDs, game times | Every cron | `site.api.espn.com/.../scoreboard` |
| ESPN Odds API | Lines, odds, opening lines | Every cron | `sports.core.api.espn.com/.../odds` |
| ESPN Injuries API | Injury status, return dates | Every cron | `site.api.espn.com/.../injuries` |
| DailyFaceoff | NHL starting goalies, stats | Every cron (NHL) | `dailyfaceoff.com/starting-goalies/` |
| Action Network | Public bet % (tickets) | Per-game (agent) | `actionnetwork.com/{sport}/public-betting` |
| SportsBettingDime | Money % (handle) for sharp detection | Per-game (agent) | `sportsbettingdime.com/{sport}/public-betting-trends` |
| WebSearch | Late scratches, GTD updates | Per-game (agent) | Dynamic |

## For LLM Assistants

When helping with this project:

1. **Automated refresh**: pg_cron handles odds + injuries + goalies every 30 min (gameday)
2. **Use the view**: Query `game_analysis_view` - aggregates odds, injuries, line movement
3. **Check new tables**: `game_goalies` for NHL starters, `injury_history` for tracking
4. **Opus-only prediction**: Each game gets a fresh Opus agent
5. **Inline research for live data**: Agent does 2-3 WebSearches per game:
   - Public betting (Action Network + SportsBettingDime)
   - Late scratches/GTD updates (supplements cron data)
6. **Log ALL decisions**: Write to `prediction_log` for BET and PASS
7. **System version**: Tag decisions with `v15-data-validation`
8. **Token efficiency**: Fresh Opus context discarded per game

## File Structure

```
supa_bet/
├── CLAUDE.md                           # This file - single source of truth
│
├── supabase/
│   └── functions/                      # Edge Functions (deployed to Supabase)
│       ├── refresh-odds/               # Odds + public betting refresh
│       ├── refresh-injuries/           # Injury scraping from ESPN HTML
│       └── settle-games/               # Game settlement + bet grading
│
└── .claude/
    ├── settings.local.json             # Local configuration
    │
    ├── skills/
    │   ├── refresh-games/SKILL.md      # Manual refresh override
    │   ├── discover-games/SKILL.md     # Find games 3-7 days out
    │   ├── predict/SKILL.md            # Lightweight orchestrator (spawns game-predictor)
    │   └── settle-bets/SKILL.md        # Results checking + bet grading
    │
    └── agents/
        ├── refresh-orchestrator/AGENT.md   # ESPN JSON API coordinator
        └── game-predictor/AGENT.md         # Opus decision maker (does inline public betting research)
```

## Known Issues & Tasks

### P0 - Critical (blocks core feature)

- [x] **Fix spread sign convention** in `refresh-odds/index.ts` - FIXED 2026-01-02 (v12)
  - Now uses `item.awayTeamOdds?.current?.pointSpread?.american` (signed) instead of `item.spread` (unsigned)
  - Historical data before 2026-01-02 has sign mismatch (deferred correction)
- [x] **Fix is_opening logic** - FIXED 2026-01-02 (v12)
  - Now checks if history exists before marking `is_opening=true`
  - Backfilled 173 games with proper opening line flags

### P1 - Important (data quality)

- [x] **Fix timezone handling** for game_time filtering - FIXED 2026-01-02
  - Added `game_time_utc` TIMESTAMPTZ column to `games` table
  - Updated `game_analysis_view` to use proper UTC calculation
  - Edge Function v12 writes to `game_time_utc`
- [x] **Add unique constraint** on placed_bets - FIXED 2026-01-02
  - Added `(game_date, away_team, home_team, bet_type, selection)` unique constraint
- [x] **Make prediction_log.game_id NOT NULL** - FIXED 2026-01-02
  - All predictions must link to a game for analytics
- [x] **Add game_id to injury_history** - FIXED 2026-01-02
  - Enables correlation of injury announcements to line movements
  - Edge Function v12 populates game_id when matching games
- [x] **Link prediction_log to outcomes** - FIXED 2026-01-03 (settle-games v3)
  - Added `result` column to `prediction_log` table (WIN/LOSS/PUSH, NULL for PASS)
  - `settle-games` Edge Function now updates `prediction_log.result` when settling bets
  - Backfilled 19 historical predictions (13 WIN, 6 LOSS)
- [x] **settle-games Edge Function team abbreviation matching** - FIXED 2026-01-03 (v2)
  - **Root cause**: `determineBetResult()` used word matching that failed for abbreviations (BKN, CLE, PHX, NYR, MIN)
  - **Fix**: Added `TEAM_ABBREVIATIONS` mapping (~80 teams: NBA, NHL, NFL, NCAAB) and `selectionMatchesTeam()` helper
  - **Affected bets**: BKN -1.5, CLE -13.5, PHX -12.5, NYR +1.5, MIN ML (manually settled before fix deployed)

### P2 - Valuable (ML prep)

- [ ] **Team name normalization** - Create canonical teams/aliases tables for reliable matching
- [ ] **Backfill spread data** - Attempt programmatic correction of pre-2026-01-02 records
- [ ] **Multi-book odds** - Track FanDuel, BetMGM, Caesars in addition to DraftKings
- [ ] **Create analytics view** showing prediction accuracy by confidence bucket
- [ ] **Track model version performance** - compare v10 vs v11 vs v12 win rates separately
- [ ] **Add feature columns** to prediction_log (opening_spread, spread_movement, etc.)

### P3 - Nice to have

- [ ] **Alerting** for settlement failures or big line moves
- [ ] **Backtesting framework** for strategy iteration
- [x] **Investigate injuries** - RESOLVED: Now using ESPN JSON injuries API via cron (not WebSearch)

## Future Enhancements

- [ ] Multi-book odds (FanDuel, BetMGM, Caesars)
- [x] NHL goalie lineup checking (via inline WebSearch)
- [ ] ML pattern matching on prediction_log
- [ ] Push notifications for significant line movements

## Audit Summary (2026-01-02)

**Overall Grade: B-** (solid foundation, one critical bug)

| Metric | Value |
|--------|-------|
| Games tracked | 293 |
| ESPN ID match rate | 92% |
| Odds history entries | 5,685 |
| Prediction log entries | 68 |
| Settled win rate | **77%** (10W/3L) |
| BET rate | 28% (19/68) |

**Key finding**: Early 77% win rate suggests real edge detection, but RLM feature broken due to sign bug.

## Version

**Current**: `v16-prediction-tracking` (2026-01-03)
- **FIXED**: `settle-games` v3 - team abbreviation matching (BKN, CLE, PHX, NYR, MIN now work)
- **FIXED**: `settle-games` v3 - now updates `prediction_log.result` when settling bets
- **NEW**: `prediction_log.result` column tracks WIN/LOSS/PUSH for BET decisions
- Edge Function: `settle-games` v3 with abbreviation mapping + prediction tracking

**Previous**: `v15-data-validation` (2026-01-02)
- **FIXED**: Spread sign convention - now uses signed `awayTeamOdds.current.pointSpread.american`
- **FIXED**: Timezone handling - new `game_time_utc` TIMESTAMPTZ column, view uses proper UTC calculation
- **FIXED**: `is_opening` logic - checks if history exists before marking (173 games backfilled)
- **NEW**: Archive tables (`odds_history_archive`, `injury_history_archive`) preserve data after cleanup
- **NEW**: `cleanup_old_games()` archives before delete, 7-day default retention
- **NEW**: `injury_history.game_id` links injuries to specific games for RLM correlation
- **NEW**: Unique constraint on `placed_bets` prevents duplicate bets
- **NEW**: `prediction_log.game_id` is now NOT NULL (required)
- Edge Function: `refresh-odds` v12 with all data validation fixes
- Sharp detection: bet% vs money% differential from Action Network + SportsBettingDime
- Split architecture: Haiku researches, Opus decides
- Per-bet-type confidence: Rates spread/ML/total 0-100 separately

**Previous**: `v14-atomic-data` (2025-12-29)
- Atomic data capture - odds + injuries + goalies fetched together
- `injury_history` table tracks when injuries are first reported + status changes
- `game_goalies` table stores NHL starting goalies from DailyFaceoff

---

*This file is the single source of truth for project documentation. Skills and agents have their own SKILL.md/AGENT.md files with implementation details.*
