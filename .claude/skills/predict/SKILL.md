---
name: predict
description: Analyze the next upcoming game and make a betting pick. Use when user says "predict games", "analyze today's slate", "make picks", or "find edges".
allowed-tools: Read, WebSearch, WebFetch, Task
---

# Predict Games (v8 - Simplified)

**Key principle:** Each game = fresh opus context. Full autonomy per game.

## Architecture

```
/predict (Lightweight Orchestrator - YOU)
    │
    ├── Step 1: Query all UNSTARTED games from Supabase
    │
    └── Step 2: FOR EACH GAME:
            └── Task(opus): game-predictor
                ├── Receives: Full game data from Supabase
                ├── Has: Full autonomy to research more if needed
                ├── Writes: prediction_log + placed_bets (if BET)
                └── Returns: Short summary

        [CONTEXT DISCARDED - next game gets fresh opus]
```

---

## Step 1: Query Games

Get all unstarted games:

```sql
SELECT *
FROM game_analysis_view
WHERE game_date >= CURRENT_DATE
  AND game_time > NOW()
ORDER BY game_time;
```

If no results: "No games to analyze. Run /refresh-games to update."

---

## Step 2: Spawn Game Predictors

For EACH game, spawn a `game-predictor` agent:

```
Task(opus, subagent_type='game-predictor'):

Game: {away_team} @ {home_team}
Sport: {sport}
Date: {game_date}
Time: {game_time}

GAME DATA FROM SUPABASE:
{Full game JSON from game_analysis_view - odds, injuries, line_movement}

Make your decision. Write to database. Return a short summary.
```

**CRITICAL:** Each game gets a FRESH agent. Do NOT batch games or pass previous game context.

---

## Step 3: Collect Summaries

Display each game-predictor's summary to the user:

```
## Predictions

### Game 1: DET @ LAL
BET: DET -2.5 @ -108 (70% conf)
Line moved: 1.5 → 2.5 | Injuries: LAL missing Reaves

### Game 2: BOS @ UTA
PASS: No edge - lines stable, no significant injuries

### Game 3: ...
```

---

## Token Budget

| Component | Tokens | Accumulates? |
|-----------|--------|--------------|
| Orchestrator base | 2k | Once |
| Per-game opus | 6-10k | **Discarded** |
| Per-game summary | ~200 | Yes |
| **10 games** | **~4k orchestrator** | Fresh per game |

---

## Notes

- Each game-predictor has full autonomy to research (WebSearch, WebFetch) if needed
- Public betting data is NOT pre-fetched; agents can research if they want
- Games can be re-predicted (no UNIQUE constraint) - useful for line movement
- Version: v8-simplified
