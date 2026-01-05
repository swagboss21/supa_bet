# Supa Bet - Sports Betting Agent System

## What This Is

A hierarchical agent system for sports betting predictions using Claude Code skills and agents. Follows the LATM (Large Language Models as Tool Makers) pattern where single-purpose agents prevent hallucinations by isolating tasks and verifying each other.

## Quick Start

```bash
# Morning briefing: cron health, overnight results, today's schedule
/status

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
                     │     EDGE FUNCTION: refresh-odds (v15)   │
                     │  1. Fetch ESPN scoreboard (event IDs)   │
                     │  2. Capture ESPN team.id → teams table  │
                     │  3. Fetch ESPN odds API (signed spread) │
                     │  4. Fetch ESPN injuries API (JSON)      │
                     │  5. Fetch DailyFaceoff goalies (NHL)    │
                     │  6. Upsert all data with team_id links  │
                     └─────────────────────────────────────────┘
                                        │
                                        ▼
                     ┌─────────────────────────────────────────┐
                     │              SUPABASE                   │
                     │  ├── teams (ESPN IDs, abbreviations)    │
                     │  ├── team_aliases (nicknames, variants) │
                     │  ├── games (+ team_id FKs)              │
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
                ├── Writes: prediction_log (single table)
                └── Returns: Short summary (~8-10k tokens)

            [CONTEXTS DISCARDED - next game fresh]

Key: Haiku researches (cheap), Opus decides (quality). ~50% token reduction.

═══════════════════════════════════════════════════════════════════
                    PHASE 3: SETTLEMENT
═══════════════════════════════════════════════════════════════════

/settle-bets → ESPN scoreboard → Match scores → Update prediction_log
                                               (result, final_score, settled_at)
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

**Transport command** (if MCP needs reconfiguring):
```bash
claude mcp add --scope project --transport http supabase "https://mcp.supabase.com/mcp?project_ref=htnhszioyydsjzkqeowx"
```

### Tables

| Table | Key Columns | Purpose |
|-------|-------------|---------|
| `teams` | espn_team_id, sport, display_name, abbreviation | Canonical team reference with ESPN IDs |
| `team_aliases` | team_id, alias, alias_type | Alternative names (nicknames, common abbrevs) |
| `games` | sport, game_date, away_team, home_team, away_team_id, home_team_id, espn_event_id, status | Core game data |
| `books` | name, short_name, priority | Sportsbook reference (DK, FD, MGM, CZR) |
| `game_odds` | game_id, book_id, spread, total, away_ml, home_ml | Current odds per book |
| `odds_history` | game_id, spread, total, open_spread, open_total, hours_before_game | Line movement snapshots |
| `game_injuries` | game_id, player_name, team, side, status | Player injury reports (linked to games) |
| `injury_history` | sport, team, player_name, status, previous_status, first_reported | Tracks when injuries announced + status changes |
| `game_goalies` | game_id, away_goalie_name, home_goalie_name, *_confirmed, *_gaa, *_sv_pct | NHL starting goalies from DailyFaceoff |
| `prediction_log` | game_id, decision, bet_type, selection, odds, result, game_context | **Unified predictions + bets** |
| `odds_history_archive` | game_snapshot (JSONB), recorded_at, spread, open_spread | Preserved odds after game cleanup |
| `injury_history_archive` | game_snapshot (JSONB), recorded_at, player_name, status | Preserved injuries after game cleanup |

> **Note**: `prediction_log` is the single source of truth for all predictions (BET and PASS).
> Use `prediction_analysis_view` for easy querying with game context.


### Views

#### `game_analysis_view` - Aggregated game data for agents

Pre-aggregates all game data into a single row per game with nested JSON:

```sql
SELECT * FROM game_analysis_view WHERE sport = 'NBA';
```

Returns columns:
- `game_id`, `sport`, `game_date`, `away_team`, `home_team`, `status`, `espn_event_id`
- `away_team_id`, `home_team_id` - FK to teams table
- `away_abbrev`, `home_abbrev` - Team abbreviations (e.g., "CLE", "NYK")
- `away_espn_id`, `home_espn_id` - ESPN team IDs for external matching
- `game_time` - UTC timestamp (TIMESTAMPTZ, for filtering)
- `game_time_local` - Eastern Time (for display)
- `odds` - Best available odds (JSON object)
- `all_odds` - All book odds (JSON array)
- `injuries` - Grouped by away/home (JSON object)
- `line_movement` - Enhanced tracking (JSON object):
  - `opening_spread`, `current_spread`, `spread_change`, `spread_direction`
  - `opening_total`, `current_total`, `total_change`
  - `hours_until_game`

#### `prediction_analysis_view` - Unified prediction data for analysis

Joins prediction_log with games and teams for easy querying:

```sql
SELECT * FROM prediction_analysis_view WHERE decision = 'BET' AND result IS NOT NULL;
```

Returns columns:
- All prediction_log columns plus:
- `sport`, `game_date`, `away_team`, `home_team`, `game_status` - from games
- `away_abbrev`, `home_abbrev` - from teams
- `max_confidence` - computed: MAX(spread_confidence, ml_confidence, total_confidence)

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

-- Win rate (from unified prediction_log)
SELECT result, COUNT(*)
FROM prediction_log
WHERE decision = 'BET' AND result IS NOT NULL
GROUP BY result;

-- Pending bets
SELECT * FROM prediction_analysis_view
WHERE decision = 'BET' AND result IS NULL AND selection IS NOT NULL;
```

## Edge Functions

All Edge Functions can be triggered manually via HTTP POST:

```bash
# Base URL
https://htnhszioyydsjzkqeowx.supabase.co/functions/v1/{function-name}
```

### `refresh-odds` - Automated Odds + Injuries + Goalies (v13)

**Atomic data capture**: Fetches odds, injuries, and goalies in a single function call so all data has the same timestamp. This enables line movement correlation with injury announcements.

**Timezone handling**: Stores `game_time` as TIMESTAMPTZ (UTC). View provides `game_time_local` for Eastern Time display.

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
3. Settles `prediction_log`: `result` (WIN/LOSS/PUSH), `final_score`, `settled_at`
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
| `status` | `/status` | Morning briefing: cron health, overnight results, today's schedule, performance |
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
2. **Use the view**: Query `game_analysis_view` for games, `prediction_analysis_view` for predictions
3. **Single table for predictions**: Write ALL predictions to `prediction_log` (BET and PASS)
4. **Unified schema**: `prediction_log` contains both decision AND bet details (selection, odds, etc.)
5. **Opus-only prediction**: Each game gets a fresh Opus agent
6. **Inline research**: Agent does WebSearches for public betting, injuries, lineups
7. **System version**: Tag decisions with `v20-unified-predictions`
8. **Token efficiency**: Fresh Opus context discarded per game
9. **Team IDs**: Use `away_abbrev`/`home_abbrev` from view for consistent team references
10. **Settlement**: `settle-games` Edge Function updates `prediction_log` directly

## File Structure

```
supa_bet/
├── CLAUDE.md                           # This file - single source of truth
│
├── supabase/
│   └── functions/                      # Edge Functions (deployed to Supabase)
│       ├── refresh-odds/               # Odds + injuries + goalies (v14)
│       ├── settle-games/               # Game settlement + bet grading (v5)
│       └── seed-teams/                 # One-time team seeding (v1)
│
└── .claude/
    ├── settings.local.json             # Local configuration
    │
    ├── skills/
    │   ├── status/SKILL.md             # Morning briefing + daily report
    │   ├── refresh-games/SKILL.md      # Manual refresh override
    │   ├── discover-games/SKILL.md     # Find games 3-7 days out
    │   ├── predict/SKILL.md            # Lightweight orchestrator
    │   └── settle-bets/SKILL.md        # Results checking + bet grading
    │
    └── agents/
        ├── refresh-orchestrator/AGENT.md   # ESPN JSON API coordinator
        ├── game-researcher/AGENT.md        # Haiku - public betting + injury research
        └── game-predictor/AGENT.md         # Opus - decision maker
```

## Known Issues & Tasks

### P0 - Critical (blocks core feature)

- [x] **Fix spread sign convention** in `refresh-odds/index.ts` - FIXED 2026-01-02 (v12)
  - Now uses `item.awayTeamOdds?.current?.pointSpread?.american` (signed) instead of `item.spread` (unsigned)
  - Historical data before 2026-01-02 has sign mismatch (deferred correction)
- [x] **Fix is_opening logic** - FIXED 2026-01-02 (v12)
  - Now checks if history exists before marking `is_opening=true`
  - Backfilled 173 games with proper opening line flags
- [x] **Fix predict skill timezone bug** - FIXED 2026-01-04, CONSOLIDATED 2026-01-04
  - **Root cause**: Two columns (`game_time` local ET, `game_time_utc` UTC) violated DRY
  - **Impact**: 19 of 29 games filtered out on 2026-01-04 despite not having started
  - **Final fix**: Consolidated to single `game_time` TIMESTAMPTZ (UTC) column, view exposes `game_time_local` for display

### P1 - Important (data quality)

- [x] **Timezone column consolidation** - FIXED 2026-01-04 (v13)
  - Dropped old `game_time` (local ET), renamed `game_time_utc` → `game_time`
  - Single `game_time` TIMESTAMPTZ column now stores UTC
  - View provides `game_time_local` for Eastern Time display
  - Edge Function v13 writes UTC only
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

- [x] **Team name normalization** - FIXED 2026-01-04 (v19)
  - Created `teams` table with ESPN IDs, abbreviations, display names
  - Created `team_aliases` table for alternative names (nicknames, common abbreviations)
  - Added `away_team_id`, `home_team_id` columns to `games`, `prediction_log`
  - Edge Function v14 captures ESPN team.id and upserts to teams table
  - `settle-games` v4 uses DB lookups instead of hardcoded TEAM_ABBREVIATIONS
  - `game_analysis_view` now includes `away_abbrev`, `home_abbrev`, `*_espn_id`
  - Dynamic NCAAB seeding - college teams added automatically on first encounter
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

## Current Stats (2026-01-04)

| Metric | Value |
|--------|-------|
| Games tracked | 350 |
| Teams normalized | 102 |
| Predictions (total) | 96 |
| Bets placed | 36 |
| Settled bets | 29 (16W-13L) |
| **Win rate** | **55.2%** |
| Pending bets | 7 |

**Note**: Historical data before 2026-01-02 has spread sign issues. Win rate is above 52.4% breakeven threshold.

## Version

**Current**: `v22-injury-history-cleanup` (2026-01-04)
- **FIXED**: `injury_history` bloat - deleted 56,773 rows (62% reduction)
- Edge Function `refresh-odds` v15 now only INSERTs on `first_reported=true` OR `status_changed=true`
- Prevents future bloat (~1,900 → ~0-50 inserts per refresh)

**Previous**: `v21-table-cleanup` (2026-01-04)
- **DROPPED**: `placed_bets` table removed after successful migration to `prediction_log`
- `prediction_log` is now the single source of truth for all predictions and bets
- 36 bets migrated, 4 legacy v4-agent bets (2025-12-16) dropped (no game records)

**Previous**: `v20-unified-predictions` (2026-01-04)
- Bet columns added to `prediction_log`: `bet_type`, `selection`, `odds`, `book`, `game_context`, `final_score`, `settled_at`
- `prediction_analysis_view` - unified view joining predictions with games and teams
- Edge Function: `settle-games` v5 now settles directly in `prediction_log`

**Previous**: `v19-team-normalization` (2026-01-04)
- `teams` table with ESPN IDs for canonical team reference (94 pro teams seeded)
- `team_aliases` table for nicknames, common abbreviations (22 aliases)
- `away_team_id`, `home_team_id` columns on `games`, `prediction_log`
- Edge Function: `refresh-odds` v14 captures ESPN team.id and links to teams table
- Edge Function: `settle-games` v4 uses DB lookups instead of hardcoded mapping

**Previous**: `v18-timezone-consolidation` (2026-01-04)
- Consolidated `game_time` + `game_time_utc` into single `game_time` TIMESTAMPTZ (UTC)
- `game_analysis_view` exposes `game_time_local` for Eastern Time display
- Pattern: Store UTC, display local - industry standard
- Edge Function: `refresh-odds` v13 writes UTC only

**Older versions** (v14-v17, 2025-12-29 to 2026-01-04):
- v17: Timezone fix patch (now consolidated into v18)
- v16: Prediction tracking, settle-games v3 with team abbreviation matching
- v15: Data validation fixes (spread sign, is_opening logic, archive tables)
- v14: Atomic data capture (odds + injuries + goalies together)

---

*This file is the single source of truth for project documentation. Skills and agents have their own SKILL.md/AGENT.md files with implementation details.*
