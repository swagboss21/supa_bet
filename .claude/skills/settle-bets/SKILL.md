---
name: settle-bets
description: Check game results and settle pending bets. Use when user says "settle bets", "check results", "update bet results", or "grade bets".
allowed-tools: WebFetch, mcp__supabase__execute_sql
---

# Settle Bets

Triggers the settle-games Edge Function to fetch final scores and update predictions with WIN/LOSS/PUSH results.

## What This Skill Does

1. Calls the `settle-games` Edge Function
2. Edge Function fetches ESPN scoreboards for final scores
3. Updates `games` table with status='final' and final_score
4. Updates `prediction_log` with result, final_score, settled_at
5. Returns settlement summary

## Process

### Step 1: Check Pending Predictions

```sql
SELECT id, game_id, bet_type, selection, odds,
       g.sport, g.game_date, g.away_team, g.home_team
FROM prediction_log pl
JOIN games g ON g.id = pl.game_id
WHERE pl.decision = 'BET'
  AND pl.result IS NULL
  AND pl.selection IS NOT NULL
ORDER BY g.game_date, g.sport;
```

### Step 2: Call Edge Function

Use WebFetch to trigger the settle-games Edge Function:

```
POST https://htnhszioyydsjzkqeowx.supabase.co/functions/v1/settle-games
Content-Type: application/json

{
  "date": "2026-01-03"  // optional, defaults to yesterday
}
```

Or settle specific sport:
```json
{
  "date": "2026-01-03",
  "sport": "NBA"
}
```

### Step 3: Review Results

After the Edge Function runs, query the results:

```sql
-- Recently settled predictions
SELECT
  g.sport, g.game_date, g.away_team, g.home_team,
  pl.selection, pl.odds, pl.result, pl.final_score
FROM prediction_log pl
JOIN games g ON g.id = pl.game_id
WHERE pl.settled_at > NOW() - INTERVAL '1 hour'
ORDER BY pl.settled_at DESC;
```

### Step 4: Win Rate Summary

```sql
-- Overall win rate
SELECT
  result,
  COUNT(*) as count,
  ROUND(100.0 * COUNT(*) / SUM(COUNT(*)) OVER(), 1) as pct
FROM prediction_log
WHERE decision = 'BET' AND result IS NOT NULL
GROUP BY result;

-- By sport
SELECT
  g.sport,
  COUNT(*) FILTER (WHERE pl.result = 'WIN') as wins,
  COUNT(*) FILTER (WHERE pl.result = 'LOSS') as losses,
  COUNT(*) FILTER (WHERE pl.result = 'PUSH') as pushes,
  ROUND(100.0 * COUNT(*) FILTER (WHERE pl.result = 'WIN') /
    NULLIF(COUNT(*) FILTER (WHERE pl.result IN ('WIN', 'LOSS')), 0), 1) as win_pct
FROM prediction_log pl
JOIN games g ON g.id = pl.game_id
WHERE pl.decision = 'BET' AND pl.result IS NOT NULL
GROUP BY g.sport;
```

## Output

Return a summary of settled predictions:

```markdown
## Bet Settlement Results (2026-01-03)

### Settled Predictions
| Sport | Game | Pick | Result | Score | Odds |
|-------|------|------|--------|-------|------|
| NBA | MIL @ CHA | Hornets +3.5 | WIN | 98-102 | -110 |
| NCAAB | DUKE @ WAKE | Wake +7.5 | LOSS | 82-71 | -110 |

### Summary
| Result | Count |
|--------|-------|
| WIN | 5 |
| LOSS | 3 |
| PUSH | 0 |
| Pending | 12 |

### Win Rate: 62.5% (5-3)
```

## Important Notes

1. **Only settle Final games** - Edge Function skips games still in progress
2. **Uses prediction_log** - Single source of truth for all predictions
3. **Team matching via teams table** - Uses ESPN IDs and abbreviations
4. **Idempotent** - Safe to run multiple times; already-settled predictions are skipped
