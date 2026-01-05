---
name: predict
description: Analyze the next upcoming game and make a betting pick. Use when user says "predict games", "analyze today's slate", "make picks", or "find edges".
allowed-tools: Read, WebSearch, WebFetch, Task, mcp__supabase__execute_sql
---

# Predict Games (v12 - Split Agents)

**Key principle:** Haiku researches, Opus decides. Fresh context per game.

## Architecture

```
/predict (Lightweight Orchestrator - YOU)
    │
    ├── Query all TODAY's games from game_analysis_view
    │
    └── FOR EACH GAME (fresh context each):
            │
            ├── Step 1: Task(haiku) → game-researcher
            │   ├── WebFetch: Action Network public betting
            │   ├── WebSearch: Injuries
            │   ├── WebSearch: Lineups/scratches
            │   ├── Returns: Research JSON
            │   └── [Context discarded]
            │
            └── Step 2: Task(opus) → game-predictor
                ├── Receives: Game data + Research JSON
                ├── Rates: spread/ML/total confidence (0-100)
                ├── Decides: BET or PASS
                ├── Writes: prediction_log (single table for all decisions)
                └── [Context discarded]

    [Next game starts with completely fresh context]
```

---

## Step 1: Query Games

Get today's unstarted games with odds:

```sql
SELECT *
FROM game_analysis_view
WHERE game_date = CURRENT_DATE
  AND status = 'scheduled'
  AND game_time > NOW()
  AND odds IS NOT NULL
  AND (odds->>'spread') IS NOT NULL
ORDER BY game_time;
```

**Filters:**
- Today only (not future days)
- Has betting odds (skip exhibitions)
- Hasn't started yet (game_time is UTC TIMESTAMPTZ)

**Columns available:**
- `game_time` - UTC timestamp (for filtering/comparisons)
- `game_time_local` - Eastern Time (for display to user)

If no results: "No games to analyze. Run /refresh-games to update."

---

## Step 2: For Each Game - Research (Haiku)

Spawn `game-researcher` agent with model=haiku:

```
Task(haiku, subagent_type='game-researcher'):

Sport: {sport}
Away Team: {away_team}
Home Team: {home_team}
Game Date: {game_date}

Research this game and return JSON with public betting, injuries, and lineups.
```

**Wait for response.** The agent returns a JSON object like:
```json
{
  "public_betting": {"spread_pct": 65, "spread_side": "home", ...},
  "injuries": {"away": [...], "home": [...]},
  "lineups": {"scratches": [...], "goalie": "..."}
}
```

---

## Step 3: For Each Game - Decision (Opus)

Spawn `game-predictor` agent with model=opus:

```
Task(opus, subagent_type='game-predictor'):

Game: {away_team} @ {home_team}
Sport: {sport}
Date: {game_date}
Time: {game_time_local}
Game ID: {game_id}

GAME DATA:
- Odds: {odds JSON}
- Line Movement: {line_movement JSON}
- Injuries (DB): {injuries JSON}

FRESH RESEARCH:
{research JSON from game-researcher}

Analyze all signals, rate confidence for spread/ML/total, make your BET/PASS decision.
Write to database. Return a short summary.
```

---

## Step 4: Collect Summaries

Display each game's result:

```
## Predictions

### Game 1: DET @ LAL (NBA)
BET: DET -2.5 @ -108
Confidence: spread=72 | ML=45 | total=30
Public: 65% LAL | Line: 1.5→2.5 | LAL missing Reaves

### Game 2: BOS @ UTA (NBA)
PASS: BOS @ UTA
Confidence: spread=42 | ML=38 | total=35
No edge - public 52/48, lines stable

### Game 3: ...
```

---

## Token Budget

| Component | Model | Tokens | Accumulates? |
|-----------|-------|--------|--------------|
| Orchestrator base | - | ~2k | Once |
| Per-game researcher | Haiku | ~3-4k | **Discarded** |
| Per-game predictor | Opus | ~8-10k | **Discarded** |
| Per-game summary | - | ~200 | Yes |

**For 10 games:**
- 10 Haiku × 4k = 40k tokens (discarded)
- 10 Opus × 9k = 90k tokens (discarded)
- Orchestrator: ~4k tokens total

**vs old architecture (1 Opus doing everything):**
- Old: 10 × 27k = 270k tokens
- New: ~134k tokens (~50% reduction)

---

## CRITICAL Rules

1. **Process ALL games** - Do NOT filter by sport, conference, or data quality
2. **Fresh context per game** - Each game gets new researcher + predictor agents
3. **Wait for research before decision** - Don't spawn predictor until researcher returns
4. **Do NOT batch games** - One game at a time through the pipeline

---

## Version

`v20-unified-predictions` - Single prediction_log table, Haiku research → Opus decision
