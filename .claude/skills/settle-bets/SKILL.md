---
name: settle-bets
description: Check game results and settle pending bets. Use when user says "settle bets", "check results", "update bet results", or "grade bets".
allowed-tools: WebFetch, WebSearch
---

# Settle Bets

Scrape final scores from ESPN and update pending bets with WIN/LOSS/PUSH results.

## What This Skill Does

1. Queries pending bets from `placed_bets` table
2. Scrapes ESPN scoreboards for final scores
3. Matches scores to bets by team names
4. Calculates WIN/LOSS/PUSH based on bet type
5. Updates `placed_bets` with result, final_score, settled_at
6. Optionally triggers game cleanup

## Process

### Step 1: Get Pending Bets

```sql
SELECT id, sport, game_date, away_team, home_team, bet_type, selection, odds
FROM placed_bets
WHERE result = 'PENDING'
ORDER BY game_date, sport;
```

### Step 2: Scrape ESPN Scoreboards

For each unique `game_date`, fetch the scoreboard:

**ESPN Scoreboard URLs** (use date format YYYYMMDD):
- NBA: `https://www.espn.com/nba/scoreboard/_/date/20251229`
- NHL: `https://www.espn.com/nhl/scoreboard/_/date/20251229`
- NFL: `https://www.espn.com/nfl/scoreboard/_/date/20251229`
- NCAAB: `https://www.espn.com/mens-college-basketball/scoreboard/_/date/20251229`

Extract from each game:
- Away team name
- Home team name
- Away score (final)
- Home score (final)
- Game status (must be "Final" to settle)

### Step 3: Match Games to Bets

Use fuzzy team name matching:
- Match by partial team name (city OR mascot)
- Example: "Milwaukee Bucks" matches "Bucks" or "Milwaukee"

```sql
-- Find matching game
SELECT * FROM pending_bets pb
WHERE pb.away_team ILIKE '%' || scraped_away || '%'
   OR pb.home_team ILIKE '%' || scraped_home || '%';
```

### Step 4: Calculate Results

**For Spread Bets:**
Parse selection to get team and spread value:
- "Charlotte Hornets +3.5" → team=Hornets, spread=+3.5
- "Milwaukee Bucks -3.5" → team=Bucks, spread=-3.5

Calculate:
```
actual_margin = (selected_team_score) - (opponent_score)
adjusted_margin = actual_margin + spread_value

IF adjusted_margin > 0 → WIN
IF adjusted_margin < 0 → LOSS
IF adjusted_margin == 0 → PUSH
```

**For Moneyline Bets:**
```
IF selected_team_score > opponent_score → WIN
IF selected_team_score < opponent_score → LOSS
IF tied → PUSH (rare, depends on sport rules)
```

**For Total Bets:**
Parse selection: "Over 228.5" or "Under 228.5"
```
actual_total = away_score + home_score

IF "Over" AND actual_total > line → WIN
IF "Over" AND actual_total < line → LOSS
IF "Under" AND actual_total < line → WIN
IF "Under" AND actual_total > line → LOSS
IF actual_total == line → PUSH
```

### Step 5: Update Placed Bets

```sql
UPDATE placed_bets
SET
  result = 'WIN',  -- or 'LOSS' or 'PUSH'
  final_score = 'Milwaukee 98 - Charlotte 102',
  settled_at = NOW()
WHERE id = 'bet-uuid';
```

### Step 6: Trigger Game Cleanup (Optional)

After settling, mark old games as stale and optionally clean up:

```sql
-- Mark stale games
SELECT mark_stale_games();

-- Clean up games older than 2 days
SELECT * FROM cleanup_old_games(2);
```

## Output

Return a summary of settled bets:

```markdown
## Bet Settlement Results (2025-12-29)

### Settled Bets
| Sport | Game | Pick | Result | Score | Odds |
|-------|------|------|--------|-------|------|
| NBA | MIL @ CHA | Hornets +3.5 | WIN | 98-102 | -110 |
| NCAAB | DUKE @ WAKE | Wake +7.5 | LOSS | 82-71 | -110 |

### Summary
| Result | Count |
|--------|-------|
| WIN | 1 |
| LOSS | 1 |
| PUSH | 0 |
| Still Pending | 2 |

### Games Cleaned Up
| Sport | Deleted |
|-------|---------|
| NBA | 11 |
| NHL | 11 |
```

## Important Notes

1. **Only settle Final games** - Skip games still in progress
2. **Team name matching** - Use partial/fuzzy matching for robustness
3. **Verify before updating** - Double-check team names match before settling
4. **Log everything** - Include final_score for audit trail
5. **Cleanup is optional** - Only run if user confirms
