# Supa Bet - Sports Betting Agent System

## What This Is

A hierarchical agent system for sports betting predictions using Claude Code skills and agents. Follows the LATM (Large Language Models as Tool Makers) pattern where single-purpose agents prevent hallucinations by isolating tasks and verifying each other.

## Quick Start

```bash
# Step 1: Refresh games + injuries from ESPN
/refresh-games

# Step 2: Analyze games and make picks
/predict
```

## Architecture

```
═══════════════════════════════════════════════════════════════════
                    PHASE 1: DATA REFRESH
═══════════════════════════════════════════════════════════════════

/refresh-games → refresh-orchestrator → ESPN scrape → Supabase
                                                      (games + injuries)

═══════════════════════════════════════════════════════════════════
                    PHASE 2: PREDICTION
═══════════════════════════════════════════════════════════════════

/predict → predict-orchestrator ─┬─→ nba-researcher ──┬─→ public-betting
                                 ├─→ nhl-researcher ──┤   social-sentiment
                                 ├─→ nfl-researcher ──┤
                                 └─→ ncaab-researcher─┘
                                          │
                                    VERIFY (LATM)
                                          │
                                   placed_bets table
```

## Supabase Setup

**Project**: `bet` (ID: `htnhszioyydsjzkqeowx`, Region: us-west-2)

### Tables

#### `games` - Today's games with odds and injuries
```sql
id UUID PRIMARY KEY
sport TEXT NOT NULL           -- 'NBA', 'NHL', 'NFL', 'NCAAB'
game_date DATE NOT NULL
game_time TIMESTAMPTZ
away_team TEXT NOT NULL
home_team TEXT NOT NULL
away_ml INTEGER               -- Away team moneyline
home_ml INTEGER               -- Home team moneyline
spread DECIMAL(4,1)           -- Point spread
total DECIMAL(5,1)            -- Over/under
injuries JSONB                -- {"away": ["Player (Status)"], "home": [...]}
book TEXT DEFAULT 'DraftKings'
status TEXT DEFAULT 'scheduled'
```

#### `placed_bets` - Betting history
```sql
id UUID PRIMARY KEY
sport TEXT NOT NULL
game_date DATE NOT NULL
away_team TEXT NOT NULL
home_team TEXT NOT NULL
bet_type TEXT NOT NULL        -- 'spread', 'total', 'moneyline'
selection TEXT NOT NULL       -- e.g., 'Knicks -2.5', 'Over 228.5'
odds INTEGER NOT NULL
confidence TEXT               -- 'low', 'medium', 'high'
reasoning TEXT
system_version TEXT           -- 'v1-tree-agent'
result TEXT DEFAULT 'PENDING' -- 'WIN', 'LOSS', 'PUSH', 'PENDING'
```

## Skills Reference

| Skill | Command | Purpose |
|-------|---------|---------|
| `refresh-games` | `/refresh-games` | Scrape ESPN for games, odds, injuries |
| `predict` | `/predict` | Analyze games and make picks |
| `public-betting` | (internal) | Search Action Network, Covers for public % |
| `social-sentiment` | (internal) | Search X, Reddit for sharp action |

## Agents Reference

| Agent | Purpose |
|-------|---------|
| `refresh-orchestrator` | Scrapes ESPN, updates Supabase with games + injuries |
| `predict-orchestrator` | Coordinates predictions, verifies picks (LATM verifier) |
| `nba-researcher` | NBA-specific research (back-to-backs, load management) |
| `nhl-researcher` | NHL-specific research (goalie matchups critical) |
| `nfl-researcher` | NFL-specific research (weather, prime time) |
| `ncaab-researcher` | NCAAB-specific research (blue blood fades) |

## Key Design Decisions

| Decision | Choice | Why |
|----------|--------|-----|
| Two-phase system | Refresh → Predict | Clean separation, no duplicate fetching |
| Injuries in Supabase | Pre-fetched | No cross-sport contamination during prediction |
| Single-task agents | Each agent does ONE thing | Prevents hallucinations |
| LATM verification | Orchestrator verifies | Cross-reference signals before betting |
| Conservative defaults | PASS when uncertain | Better to miss edges than make bad bets |

## Pass Criteria (v1)

- **PASS** if conflicting signals between sources
- **PASS** if confidence < 60%
- **PASS** if public betting data unavailable
- **BET** if clear public fade opportunity (>65% public + sharp opposite)

## Data Sources

| Source | Used For |
|--------|----------|
| ESPN Odds | Games, lines, odds |
| ESPN Injuries | Injury reports |
| Action Network | Public betting %, sharp reports |
| Covers | Consensus picks |
| X/Twitter | Sharp commentary |
| Reddit r/sportsbook | Community sentiment |

## For LLM Assistants

When helping with this project:

1. **Two phases**: Always refresh before predict
2. **Injuries are local**: Don't search for injuries during prediction
3. **Single-task agents**: Each agent only handles its sport
4. **PASS is okay**: Conservative betting is the goal
5. **Log everything**: All picks need reasoning
6. **System version**: Tag picks with version for tracking

## File Structure

```
.claude/
├── skills/
│   ├── refresh-games/SKILL.md      # ESPN scraping entry
│   ├── predict/SKILL.md            # Prediction entry
│   ├── public-betting/SKILL.md     # Public % research
│   └── social-sentiment/SKILL.md   # Sharp/social research
│
└── agents/
    ├── refresh-orchestrator/AGENT.md
    ├── predict-orchestrator/AGENT.md
    ├── nba-researcher/AGENT.md
    ├── nhl-researcher/AGENT.md
    ├── nfl-researcher/AGENT.md
    └── ncaab-researcher/AGENT.md
```

## Version History

- **v1-tree-agent**: Initial hierarchical agent system with LATM verification
