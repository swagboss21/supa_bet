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
                     │     EDGE FUNCTION: refresh-odds         │
                     │  1. Fetch ESPN scoreboard (event IDs)   │
                     │  2. Fetch ESPN odds API (JSON)          │
                     │  3. Upsert game_odds + odds_history     │
                     └─────────────────────────────────────────┘
                                        │
                                        ▼
                     ┌─────────────────────────────────────────┐
                     │              SUPABASE                   │
                     │  ├── games (+ espn_event_id)            │
                     │  ├── game_odds (current lines)          │
                     │  ├── odds_history (line movement)       │
                     │  └── game_injuries                      │
                     └─────────────────────────────────────────┘

═══════════════════════════════════════════════════════════════════
                    MANUAL OVERRIDE: /refresh-games
═══════════════════════════════════════════════════════════════════

/refresh-games → refresh-orchestrator → Edge Function OR ESPN JSON APIs
                                        (force refresh, bypass schedule)

═══════════════════════════════════════════════════════════════════
                    PREDICTION (Isolated Opus per Game)
═══════════════════════════════════════════════════════════════════

/predict (Lightweight Orchestrator)
    │
    ├── Query all UNSTARTED games from Supabase
    │
    └── FOR EACH GAME:
            └── Task(opus): game-predictor
                ├── Receives: Full game data from Supabase
                ├── Has: Full autonomy (can research more if needed)
                ├── Writes: prediction_log + placed_bets (if BET)
                └── Returns: Short summary

            [CONTEXT DISCARDED - next game gets fresh opus]

Key: Each game = fresh context. Full autonomy to research if needed.

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

#### `games` - Games table (core table)

```sql
CREATE TABLE games (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  created_at TIMESTAMPTZ DEFAULT NOW(),

  -- Game info
  sport TEXT NOT NULL,              -- 'NBA', 'NHL', 'NFL', 'NCAAB'
  game_date DATE NOT NULL,
  game_time TIMESTAMPTZ,
  away_team TEXT NOT NULL,
  home_team TEXT NOT NULL,
  espn_event_id TEXT,               -- ESPN event ID for JSON API lookups

  -- Status
  status TEXT DEFAULT 'scheduled',
  final_score TEXT,

  UNIQUE (sport, game_date, away_team, home_team)
);

-- Index for ESPN event ID lookups
CREATE INDEX idx_games_espn_event ON games(espn_event_id) WHERE espn_event_id IS NOT NULL;
```

> **Note**: `espn_event_id` links to ESPN JSON APIs for structured odds data. All odds are stored in `game_odds` table (not in games table).

#### `books` - Sportsbook reference table

```sql
CREATE TABLE books (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  created_at TIMESTAMPTZ DEFAULT NOW(),
  name TEXT NOT NULL UNIQUE,        -- 'DraftKings', 'FanDuel', etc.
  short_name TEXT NOT NULL UNIQUE,  -- 'DK', 'FD', 'MGM', 'CZR'
  is_active BOOLEAN DEFAULT TRUE,
  priority INTEGER DEFAULT 100      -- Lower = preferred
);

-- Seeded with Big 4:
-- DraftKings (DK), FanDuel (FD), BetMGM (MGM), Caesars (CZR)
```

#### `game_injuries` - Player injuries linked to games

```sql
CREATE TABLE game_injuries (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW(),
  game_id UUID NOT NULL REFERENCES games(id) ON DELETE CASCADE,
  player_name TEXT NOT NULL,
  team TEXT NOT NULL,
  side TEXT NOT NULL,               -- 'away' or 'home'
  position TEXT,
  status TEXT NOT NULL,             -- 'OUT', 'GTD', 'Questionable', 'Doubtful', 'Probable'
  injury_type TEXT,
  injury_description TEXT,
  impact_level TEXT,                -- 'critical', 'high', 'medium', 'low'
  is_starter BOOLEAN DEFAULT FALSE,
  source TEXT DEFAULT 'ESPN',
  UNIQUE(game_id, player_name, team)
);
```

#### `game_odds` - Multi-book odds per game

```sql
CREATE TABLE game_odds (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW(),
  game_id UUID NOT NULL REFERENCES games(id) ON DELETE CASCADE,
  book_id UUID NOT NULL REFERENCES books(id),
  away_ml INTEGER,
  home_ml INTEGER,
  spread DECIMAL(4,1),
  away_spread_odds INTEGER DEFAULT -110,
  home_spread_odds INTEGER DEFAULT -110,
  total DECIMAL(5,1),
  over_odds INTEGER DEFAULT -110,
  under_odds INTEGER DEFAULT -110,
  is_current BOOLEAN DEFAULT TRUE,
  fetched_at TIMESTAMPTZ DEFAULT NOW(),
  UNIQUE(game_id, book_id)
);
```

#### `odds_history` - Line movement tracking

```sql
CREATE TABLE odds_history (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  recorded_at TIMESTAMPTZ DEFAULT NOW(),
  game_id UUID NOT NULL REFERENCES games(id) ON DELETE CASCADE,
  book_id UUID NOT NULL REFERENCES books(id),

  -- Current odds at snapshot time
  away_ml INTEGER,
  home_ml INTEGER,
  spread DECIMAL(4,1),
  away_spread_odds INTEGER,
  home_spread_odds INTEGER,
  total DECIMAL(5,1),
  over_odds INTEGER,
  under_odds INTEGER,

  -- Opening line tracking (from ESPN API)
  is_opening BOOLEAN DEFAULT FALSE,
  open_spread DECIMAL(4,1),         -- Opening spread from ESPN
  open_total DECIMAL(5,1),          -- Opening total from ESPN
  open_away_ml INTEGER,             -- Opening away ML
  open_home_ml INTEGER,             -- Opening home ML

  -- Metadata
  source TEXT DEFAULT 'espn_api',   -- 'espn_api', 'webfetch', 'the_odds_api'
  hours_before_game DECIMAL(6,2),   -- Hours until game when snapshot taken
  refresh_tier TEXT,                -- 'opening', '72hr', '24hr', '6hr', 'gameday'
  significant_move BOOLEAN DEFAULT FALSE,  -- Auto-detected significant movement
  market_event TEXT                 -- 'opening', 'injury_news', 'line_move'
);

-- Trigger auto-detects significant movements (>0.5 spread, >1.0 total, >15 ML)
```

#### `placed_bets` - Betting history (immutable ground truth)

```sql
CREATE TABLE placed_bets (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  created_at TIMESTAMPTZ DEFAULT NOW(),

  -- Game info
  sport TEXT NOT NULL,
  game_date DATE NOT NULL,
  game_time TIMESTAMPTZ,
  away_team TEXT NOT NULL,
  home_team TEXT NOT NULL,

  -- The bet
  bet_type TEXT NOT NULL,           -- 'spread', 'total', 'moneyline'
  selection TEXT NOT NULL,          -- e.g., 'Knicks -2.5', 'Over 228.5', 'Bucks ML'
  odds INTEGER NOT NULL,            -- American odds
  book TEXT,

  -- Tracking
  confidence TEXT,                  -- 'low', 'medium', 'high'
  reasoning TEXT,
  system_version TEXT,              -- 'v3-data-lifecycle'
  game_context JSONB,               -- Snapshot of game state at bet time

  -- Results
  result TEXT DEFAULT 'PENDING',    -- 'WIN', 'LOSS', 'PUSH', 'PENDING'
  final_score TEXT,
  settled_at TIMESTAMPTZ,

  -- Money (optional)
  stake DECIMAL(10,2),
  profit DECIMAL(10,2)
);
```

> **`game_context`**: Captures full game state at bet time for historical learning:
> ```json
> {
>   "odds": {"spread": 3.5, "total": 228.5, "away_ml": -155},
>   "injuries": {"away": [...], "home": [...]},
>   "signals": {"public_pct": 68, "sharp_side": "home"},
>   "line_movement": {"opening_spread": 4.0}
> }
> ```

#### `prediction_log` - ALL decisions (BET + PASS) for learning

```sql
CREATE TABLE prediction_log (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  created_at TIMESTAMPTZ DEFAULT NOW(),

  -- Game reference
  game_id UUID REFERENCES games(id) ON DELETE SET NULL,
  sport TEXT NOT NULL,
  game_date DATE NOT NULL,
  away_team TEXT NOT NULL,
  home_team TEXT NOT NULL,

  -- Decision
  decision TEXT NOT NULL CHECK (decision IN ('BET', 'PASS')),

  -- Structured signals (queryable for ML)
  public_pct DECIMAL(4,1),           -- e.g., 68.5
  public_side TEXT,                   -- 'away', 'home', 'split'
  sharp_detected BOOLEAN DEFAULT FALSE,
  sharp_side TEXT,                    -- 'away', 'home'
  rlm_detected BOOLEAN DEFAULT FALSE, -- Reverse line movement
  goalie_edge TEXT,                   -- 'away', 'home' (NHL only)

  -- Odds at decision time
  spread DECIMAL(4,1),
  total DECIMAL(5,1),
  away_ml INTEGER,
  home_ml INTEGER,

  -- Link to bet if placed
  placed_bet_id UUID REFERENCES placed_bets(id) ON DELETE SET NULL,

  -- Reasoning
  reasoning TEXT NOT NULL,
  confidence DECIMAL(3,2),            -- 0.00-1.00

  -- Efficiency tracking
  tokens_used INTEGER,
  prediction_number INTEGER DEFAULT 1,  -- Tracks re-analyses of same game
  decision_model TEXT DEFAULT 'opus',
  system_version TEXT DEFAULT 'v7-isolated'

  -- NOTE: No UNIQUE on game_id - allows re-prediction as lines move
);
```

> **Why log PASS decisions?** The system learns from patterns. Knowing WHY we passed (e.g., "public 50/50, no edge") is as valuable as knowing why we bet.

#### `game_public_betting` - Public betting % from multiple sources

```sql
CREATE TABLE game_public_betting (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  created_at TIMESTAMPTZ DEFAULT NOW(),
  game_id UUID NOT NULL REFERENCES games(id) ON DELETE CASCADE,

  -- Source tracking (store ALL sources for future arbitrage)
  source TEXT NOT NULL,  -- 'covers', 'action_network', 'vegas_insider'
  source_priority INTEGER DEFAULT 100,  -- Lower = more reliable (covers=10)

  -- Spread betting
  spread_public_pct DECIMAL(4,1),
  spread_public_side TEXT,  -- 'away' or 'home'
  spread_money_pct DECIMAL(4,1),
  spread_money_side TEXT,

  -- Moneyline betting
  ml_public_pct DECIMAL(4,1),
  ml_public_side TEXT,
  ml_money_pct DECIMAL(4,1),
  ml_money_side TEXT,

  -- Total betting
  total_public_pct DECIMAL(4,1),
  total_money_pct DECIMAL(4,1),

  -- Metadata
  fetched_at TIMESTAMPTZ DEFAULT NOW(),
  raw_data JSONB,

  UNIQUE(game_id, source)
);
```

> **Note**: This table is reserved for a future pipeline. Currently empty because Action Network and Covers use JavaScript rendering (can't be scraped server-side). The `game-predictor` agent can research public betting via WebSearch if needed.

### Schema Relationships

```
books (reference)
├── id, name, short_name, priority

games
├── id, sport, game_date, away_team, home_team, espn_event_id
├── UNIQUE(sport, game_date, away_team, home_team)
│
├──► game_injuries (game_id FK)
│    └── player_name, team, side, status, impact_level
│
├──► game_odds (game_id FK + book_id FK)
│    └── away_ml, home_ml, spread, total, is_current
│
├──► odds_history (game_id FK + book_id FK)
│    └── recorded_at, is_opening, open_spread, open_total, significant_move
│
├──► prediction_log (game_id FK, multiple per game allowed)
│    └── decision, public_pct, sharp_detected, reasoning, prediction_number
│
└──► game_public_betting (game_id FK)
     └── source, spread_public_pct, ml_public_pct, total_public_pct

placed_bets (standalone - immutable record)
└──► prediction_log.placed_bet_id (optional link when decision=BET)
```

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
  - `snapshots` (count), `significant_moves` (count)
  - `hours_until_game`, `last_update`
- `public_betting` - Pre-scraped from Covers/Action Network (JSON array):
  - `source`, `priority`, `spread_pct`, `spread_side`, `ml_pct`, `total_pct`

### Functions

#### `cleanup_old_games(days_old INTEGER)`

Removes games older than X days with terminal status. Cascades to delete odds, injuries, lineups, history.

```sql
SELECT * FROM cleanup_old_games(2);  -- Returns deleted count per sport
```

#### `mark_stale_games()`

Marks games from past dates still marked as 'scheduled' as 'stale'.

```sql
SELECT mark_stale_games();  -- Returns count of games marked stale
```

### Example SQL Queries

```sql
-- Get today's games
SELECT * FROM games
WHERE game_date = CURRENT_DATE
ORDER BY sport, game_time;

-- Get games with injuries
SELECT g.*,
  json_agg(gi.*) FILTER (WHERE gi.id IS NOT NULL) AS injuries
FROM games g
LEFT JOIN game_injuries gi ON g.id = gi.game_id
WHERE g.game_date = CURRENT_DATE AND g.sport = 'NBA'
GROUP BY g.id;

-- Best odds across all books
SELECT g.away_team || ' @ ' || g.home_team AS matchup,
  b.short_name AS book,
  go.away_ml, go.home_ml, go.spread, go.total
FROM games g
JOIN game_odds go ON g.id = go.game_id AND go.is_current = TRUE
JOIN books b ON go.book_id = b.id
WHERE g.game_date = CURRENT_DATE
ORDER BY g.game_time, g.away_team;

-- Line movement for a game
SELECT oh.recorded_at, b.short_name, oh.spread, oh.total,
  oh.spread - LAG(oh.spread) OVER (ORDER BY oh.recorded_at) AS spread_move
FROM odds_history oh
JOIN books b ON oh.book_id = b.id
WHERE oh.game_id = 'some-game-uuid'
ORDER BY oh.recorded_at;

-- Insert a pick
INSERT INTO placed_bets (
  sport, game_date, away_team, home_team,
  bet_type, selection, odds, confidence, reasoning, system_version
)
VALUES (
  'NBA', '2025-12-29', 'Milwaukee Bucks', 'Charlotte Hornets',
  'moneyline', 'Bucks ML', -155, 'medium', 'Road favorite with value', 'v1-tree-agent'
);

-- Calculate win rate
SELECT
  result,
  COUNT(*) as count,
  ROUND(COUNT(*) * 100.0 / SUM(COUNT(*)) OVER (), 1) as pct
FROM placed_bets
WHERE result != 'PENDING'
GROUP BY result;
```

## Edge Functions

All Edge Functions can be triggered manually via HTTP POST:

```bash
# Base URL
https://htnhszioyydsjzkqeowx.supabase.co/functions/v1/{function-name}
```

### `refresh-odds` - Automated Odds Refresh

Fetches ESPN JSON APIs and updates odds in Supabase.

**Endpoint**: `https://htnhszioyydsjzkqeowx.supabase.co/functions/v1/refresh-odds`

**Parameters**:
```json
{
  "tier": "opening" | "72hr" | "24hr" | "6hr" | "gameday" | "all",
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
1. Fetches ESPN scoreboard for event IDs
2. Fetches ESPN odds API per event (structured JSON, not HTML)
3. Upserts to `game_odds` (current odds)
4. Inserts to `odds_history` (line movement tracking)

### `refresh-injuries` - Injury Scraper

Scrapes ESPN injury pages (HTML) and links injuries to upcoming games.

**Endpoint**: `https://htnhszioyydsjzkqeowx.supabase.co/functions/v1/refresh-injuries`

**Parameters**:
```json
{
  "sport": "NBA" | "NHL" | "NFL" | "NCAAB"  // optional, defaults to all
}
```

**Manual trigger**:
```bash
curl -X POST https://htnhszioyydsjzkqeowx.supabase.co/functions/v1/refresh-injuries
```

**What it does**:
1. Fetches ESPN injury pages (HTML scraping)
2. Parses player names, status, injury type
3. Matches injuries to upcoming games (next 7 days)
4. Upserts to `game_injuries` table

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
| `refresh-injuries` | Every 2 hours | `refresh-injuries` | Update injury reports |
| `settle-games` | 8pm/10pm/midnight UTC | `settle-games` | Settlement checks |

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

**Sport paths**:
- NBA: `basketball/nba`
- NHL: `hockey/nhl`
- NFL: `football/nfl`
- NCAAB: `basketball/mens-college-basketball`

## Skills Reference

| Skill | Command | Purpose |
|-------|---------|---------|
| `refresh-games` | `/refresh-games` | Manual override to refresh games (calls Edge Function) |
| `discover-games` | `/discover-games` | Find games 3-7 days out for opening lines |
| `predict` | `/predict` | Lightweight orchestrator - spawns fresh opus per game |
| `settle-bets` | `/settle-bets` | Check results and grade pending bets |

## Agents Reference

| Agent | Purpose |
|-------|---------|
| `refresh-orchestrator` | Calls Edge Function or ESPN JSON APIs, updates Supabase |
| `game-predictor` | Fresh opus per game - full autonomy on analysis, writes BET/PASS to database |

## Key Design Decisions

| Decision | Choice | Why |
|----------|--------|-----|
| Two-phase system | Refresh → Predict | Clean separation, no duplicate fetching |
| Injuries in Supabase | Pre-fetched | No cross-sport contamination during prediction |
| Fresh opus per game | Task(opus) agent | Each game gets clean context, no accumulation |
| Public betting | Agent autonomy | No reliable server-side source; agent can WebSearch if needed |
| Log ALL decisions | prediction_log table | Learning requires knowing why we passed, not just bet |
| Full autonomy | No rigid rules | Claude can research more if needed, find obscure edges |
| Conservative defaults | PASS when uncertain | Better to miss edges than make bad bets |
| Re-prediction allowed | No UNIQUE on game_id | Can re-analyze as lines move |

## Decision Philosophy

**Claude's reasoning backed by verified data, not rigid rules.**

Reference patterns (starting points, not constraints):
- Public >65% + sharp opposite → Fade opportunity
- Public + key injury on public side → Fade with conviction
- NHL backup goalie on public side → Goalie edge

Claude's freedom:
- Can find edges not in patterns above
- Can BET even if sharp/public aligned (if reasoning supports it)
- Can PASS on "obvious" patterns if something feels off

Conservative default: **PASS when no clear edge** - but reason with whatever signals are available

## Data Sources

| Source | Used For | URL |
|--------|----------|-----|
| ESPN Scoreboard API | Event IDs, game times | `site.api.espn.com/apis/site/v2/sports/{sport}/{league}/scoreboard` |
| ESPN Odds API | Lines, odds, opening lines | `sports.core.api.espn.com/.../events/{id}/.../odds` |
| ESPN Injuries (HTML) | Injury reports | espn.com/{sport}/injuries |
| Action Network | Public betting %, sharp reports | actionnetwork.com |
| Covers | Consensus picks, public % | covers.com |
| X/Twitter | Sharp commentary | x.com |
| Reddit | Community sentiment | reddit.com/r/sportsbook |

## For LLM Assistants

When helping with this project:

1. **Automated refresh**: pg_cron handles odds updates automatically (every 30 min gameday)
2. **Use the view**: Query `game_analysis_view` instead of complex JOINs
3. **Data is local**: Odds and injuries are in Supabase (pre-fetched by Edge Functions)
4. **Fresh opus per game**: `/predict` spawns a fresh `game-predictor` agent for each game
5. **Full autonomy**: Each game-predictor can use Supabase data or research more (WebSearch)
6. **Public betting**: Not pre-fetched (no reliable source). Agent can WebSearch if needed.
7. **Log ALL decisions**: Write to `prediction_log` for both BET and PASS (learning needs both)
8. **System version**: Tag decisions with `v8-simplified` for tracking
9. **Re-prediction OK**: Games can be re-analyzed (no UNIQUE constraint) - useful for line movement
10. **Token budget**: ~4k orchestrator for 10 games (summaries only accumulate)

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
        └── game-predictor/AGENT.md         # Fresh opus per game, full autonomy
```

## Migration Status

### Phase 1: Migrate Existing Data ✅ COMPLETE (2025-12-29)
Legacy odds migrated from `games` table to normalized structure:
- `game_odds`: 29 rows (DraftKings odds)
- `odds_history`: 29 rows (opening lines)

### Phase 2: Update Agents ✅ COMPLETE (2025-12-29)
- [x] `refresh-orchestrator` - Now writes to `game_odds`, `odds_history`, `game_injuries`
- [x] `predict-orchestrator` - Now uses `game_analysis_view` + saves `game_context`

### Phase 3: Data Lifecycle ✅ COMPLETE (2025-12-29)
- [x] `game_context` JSONB column in `placed_bets` for historical learning
- [x] `game_analysis_view` for simplified agent data access
- [x] `cleanup_old_games()` + `mark_stale_games()` functions
- [x] `/settle-bets` skill for results checking

### Phase 4: Automated Data Pipeline ✅ COMPLETE (2025-12-30)
- [x] ESPN JSON APIs (structured data, no HTML scraping for odds)
- [x] `espn_event_id` column in `games` table
- [x] `refresh_schedules` table for tier-based refresh tracking
- [x] Enhanced `odds_history` with opening lines, source, hours_before_game
- [x] `refresh-odds` Edge Function deployed
- [x] `refresh-injuries` Edge Function deployed (ESPN HTML scraping)
- [x] `settle-games` Edge Function deployed (updates games + placed_bets)
- [x] pg_cron + pg_net extensions enabled
- [x] 6 scheduled cron jobs for automated refresh
- [x] `game_analysis_view` enhanced with line movement tracking
- [x] `/discover-games` skill for finding games 3-7 days out

### Phase 4.5: Schema Cleanup ✅ COMPLETE (2025-12-30)
- [x] Dropped legacy odds columns from `games` table (`away_ml`, `home_ml`, `spread`, `total`, `book`)
- [x] All odds now exclusively in `game_odds` table
- [x] Updated CLAUDE.md documentation

### Phase 5: Simplification ✅ COMPLETE (2025-12-30)
- [x] Dropped unused tables: `game_lineups`, `refresh_schedules`
- [x] Consolidated 5 agents → 1 unified `sports-researcher`
- [x] Single game per `/predict` invocation (fresh context, no batch)
- [x] Consistent reasoning format for future pattern matching
- [x] Updated `game_analysis_view` (removed goalies, refresh_info)

### Phase 5.5: Timezone Fix ✅ COMPLETE (2025-12-30)
- [x] Fixed `refresh-odds` Edge Function: `game_date` now uses US Eastern time (not UTC)
- [x] Added `getLocalDate()` helper using `Intl.DateTimeFormat` with `America/New_York`
- [x] Corrected 128 existing games with wrong dates via SQL update
- [x] Games now store correct local date for historical tracking

### Phase 6: Efficient Prediction ✅ COMPLETE (2025-12-30)
- [x] Created `prediction_log` table for ALL decisions (BET + PASS)
- [x] Haiku→Opus flow: haiku for cheap research, opus for reasoning
- [x] Parallel haiku agents for public betting + sharp action
- [x] 87% token reduction (39k → ~5k per game)
- [x] Flexible decision philosophy (guidelines not rigid rules)
- [x] Updated `/predict` skill for efficient flow
- [x] Structured signals (public_pct, sharp_detected) for future ML

### Phase 7: Isolated Prediction ✅ COMPLETE (2025-12-30)
- [x] Fresh opus agent per game (no context accumulation)
- [x] `game_public_betting` table for pre-scraped public betting data
- [x] `game-predictor` agent with full autonomy
- [x] Removed UNIQUE on prediction_log.game_id (allows re-prediction)
- [x] Added `prediction_number` column for tracking re-analyses
- [x] Updated `game_analysis_view` with public_betting column
- [x] Edge Function scrapes Covers for public betting (placeholder for HTML parsing)
- [x] Deleted old files: sports-researcher, public-betting, social-sentiment skills

### Phase 8: Future Enhancements
- [x] ~~Implement Covers HTML parsing in Edge Function~~ (BLOCKED: Covers uses JS rendering)
- [x] ~~Add Action Network scraping~~ (BLOCKED: Action Network uses JS rendering)
- [ ] Multi-book odds (FanDuel, BetMGM, Caesars via OddsJam/Covers)
- [ ] Lineup scraper (RotoWire, DailyFaceoff for NHL goalies)
- [ ] Push notifications for significant line movements
- [ ] Historical backfill of odds data
- [ ] ML pattern matching on prediction_log
- [ ] Find server-rendered public betting source (or API)

## Version History

| Version | Date | Description |
|---------|------|-------------|
| `v1-tree-agent` | 2025-12-28 | Initial hierarchical agent system with LATM verification |
| `v2-data-layer` | 2025-12-29 | Normalized schema: `game_odds`, `odds_history`, `game_injuries` tables active |
| `v3-data-lifecycle` | 2025-12-29 | Game lifecycle: `game_analysis_view`, `game_context`, `/settle-bets`, cleanup functions |
| `v4-automation` | 2025-12-30 | Automated pipeline: ESPN JSON APIs, Edge Function, pg_cron scheduling, line movement tracking |
| `v5-simplified` | 2025-12-30 | Simplified agents: 5→1 unified researcher, one game per invocation, dropped dead schema |
| `v5.5-timezone` | 2025-12-30 | Fixed Edge Function timezone: game_date now uses US Eastern time for accurate historical tracking |
| `v6-efficient` | 2025-12-30 | Haiku→Opus flow: 87% token reduction, `prediction_log` for all decisions, flexible reasoning philosophy |
| `v7-isolated` | 2025-12-30 | Fresh opus per game: no context accumulation, `game_public_betting` table, full autonomy, re-prediction allowed |
| `v7.1-agent-fetch` | 2025-12-30 | Public betting fetched by game-predictor agent (Action Network uses JS, can't pre-scrape) |
| `v7.2-optimized` | 2025-12-30 | Two-phase /predict: parallel haiku fetchers → Supabase, then fresh opus per game. 80% fewer fetches. |
| `v8-simplified` | 2025-12-31 | Removed broken public-betting-fetcher. Single-phase /predict. Agent autonomy for research. |

---

*This file is the single source of truth for project documentation. Skills and agents have their own SKILL.md/AGENT.md files with implementation details.*
