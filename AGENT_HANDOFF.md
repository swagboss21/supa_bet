# Agent Handoff - Betting System

## Project Goal
Build agent scripts that read games/odds from Supabase and make moneyline picks.

## Supabase Setup (Already Complete)

**Project**: `bet` (ID: `htnhszioyydsjzkqeowx`, Region: us-west-2)

### Tables

#### `games` - Today's games with odds
```sql
-- Schema
id UUID PRIMARY KEY
created_at TIMESTAMPTZ
sport TEXT NOT NULL           -- 'NBA', 'NHL', 'NFL', 'NCAAB'
game_date DATE NOT NULL
game_time TIMESTAMPTZ
away_team TEXT NOT NULL
home_team TEXT NOT NULL
away_ml INTEGER               -- Away team moneyline (e.g., -155, +130)
home_ml INTEGER               -- Home team moneyline
spread DECIMAL(4,1)           -- Point spread (positive = home favored)
total DECIMAL(5,1)            -- Over/under
book TEXT DEFAULT 'DraftKings'
status TEXT DEFAULT 'scheduled'
final_score TEXT
```

#### `placed_bets` - Betting history
```sql
-- Schema
id UUID PRIMARY KEY
created_at TIMESTAMPTZ
sport TEXT NOT NULL
game_date DATE NOT NULL
game_time TIMESTAMPTZ
away_team TEXT NOT NULL
home_team TEXT NOT NULL
bet_type TEXT NOT NULL        -- 'spread', 'total', 'moneyline'
selection TEXT NOT NULL       -- e.g., 'Knicks -2.5', 'Over 228.5', 'Bucks ML'
odds INTEGER NOT NULL         -- American odds
book TEXT
confidence TEXT               -- 'low', 'medium', 'high'
reasoning TEXT
system_version TEXT           -- Track which agent/version made the pick
result TEXT DEFAULT 'PENDING' -- 'WIN', 'LOSS', 'PUSH', 'PENDING'
final_score TEXT
settled_at TIMESTAMPTZ
stake DECIMAL(10,2)
profit DECIMAL(10,2)
```

### Current Data (Dec 29, 2025)
- **NBA**: 11 games with full odds
- **NHL**: 11 games with full odds
- **NFL**: 1 game (Rams @ Falcons)
- **NCAAB**: 7 games with odds (many more without ML odds)

## Agent Script Requirements

1. **Read from Supabase**: Query `games` table for today's slate
2. **Apply prediction logic**: Analyze games, decide bet or pass
3. **Write picks to Supabase**: Insert into `placed_bets` table

### Example Queries

```sql
-- Get today's games
SELECT * FROM games WHERE game_date = CURRENT_DATE ORDER BY sport, game_time;

-- Get games by sport
SELECT * FROM games WHERE sport = 'NBA' AND game_date = CURRENT_DATE;

-- Insert a pick
INSERT INTO placed_bets (sport, game_date, away_team, home_team, bet_type, selection, odds, confidence, reasoning, system_version)
VALUES ('NBA', '2025-12-29', 'Milwaukee Bucks', 'Charlotte Hornets', 'moneyline', 'Bucks ML', -155, 'medium', 'Road favorite with value', 'v1-agent');
```

## Connection Details

The Supabase MCP is already connected. Use:
- `mcp__supabase__execute_sql` for queries
- `mcp__supabase__list_tables` to see schema
- Project ID: `htnhszioyydsjzkqeowx`

## Data Source
Games and odds scraped from ESPN (DraftKings odds). Can be refreshed by fetching:
- `espn.com/nba/odds`
- `espn.com/nfl/odds`
- `espn.com/nhl/odds`
- `espn.com/mens-college-basketball/odds`

## Next Steps for Agent Scripts
1. Create a script structure (TypeScript/Python/etc.)
2. Connect to Supabase
3. Implement prediction logic for moneylines
4. Output picks to `placed_bets` table
5. Allow passing on games (no forced bets)
