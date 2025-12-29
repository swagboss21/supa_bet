# Betting Data Lake — Context Document

## Goal
Build a permanent betting data foundation in Supabase that outlives all prediction system iterations.

## Priority 1: Placed Bets History (Today)
The single most important table. Your actual betting record — immutable ground truth.

### Schema: `placed_bets`

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
  bet_type TEXT NOT NULL,
  selection TEXT NOT NULL,
  odds INTEGER NOT NULL,
  book TEXT,
  
  -- Tracking
  confidence TEXT,
  reasoning TEXT,
  system_version TEXT,
  
  -- Results
  result TEXT DEFAULT 'PENDING',
  final_score TEXT,
  settled_at TIMESTAMPTZ,
  
  -- Money (optional)
  stake DECIMAL(10,2),
  profit DECIMAL(10,2)
);

CREATE INDEX idx_placed_bets_sport ON placed_bets(sport);
CREATE INDEX idx_placed_bets_result ON placed_bets(result);
CREATE INDEX idx_placed_bets_game_date ON placed_bets(game_date);
```

### Field Reference

| Field | Type | Example | Notes |
|-------|------|---------|-------|
| sport | TEXT | "NBA", "NCAAB" | Required |
| game_date | DATE | "2025-12-16" | Required |
| game_time | TIMESTAMPTZ | "2025-12-16T01:30:00Z" | Optional |
| away_team | TEXT | "San Antonio Spurs" | Required |
| home_team | TEXT | "New York Knicks" | Required |
| bet_type | TEXT | "spread", "total", "moneyline" | Required |
| selection | TEXT | "Knicks -2.5", "Over 228.5" | Required |
| odds | INTEGER | -110, +150 | American odds |
| book | TEXT | "DraftKings" | Nullable |
| confidence | TEXT | "low", "medium", "high" | Nullable |
| reasoning | TEXT | "Public heavy on..." | Nullable |
| system_version | TEXT | "v4-agent", "v5-mcp" | Track which system made pick |
| result | TEXT | "PENDING", "WIN", "LOSS", "PUSH" | Default PENDING |
| final_score | TEXT | "Knicks 124, Spurs 113" | Nullable |
| settled_at | TIMESTAMPTZ | When result was recorded | Nullable |
| stake | DECIMAL | 100.00 | Nullable |
| profit | DECIMAL | 90.91 (for -110 win) | Nullable |

---

## Data to Import

Noah has historical bet data from multiple prediction methods to migrate into this table. Formats TBD.

Current `history.json` has 4 picks from Dec 16 (all WINs):
- Knicks -2.5 ✓
- DePaul/St. John's Under 150 ✓
- Tennessee/Louisville Under 157.5 ✓
- Dayton -6.5 ✓

---

## Future Tables (Not Yet)

Once `placed_bets` is solid:

| Table | Purpose |
|-------|---------|
| `games` | Scraped game schedules |
| `odds` | Historical odds snapshots |
| `books` | Sportsbook reference |
| `v_best_lines` | View for line shopping |

---

## NOT in Scope Yet

- MCP server for external access
- REST API
- Agent skills
- Predictions/models
- Scrapers

---

## Next Steps

1. Create `placed_bets` table in Supabase
2. Import historical bet data
3. Pick a sport/league to build scrapers for
