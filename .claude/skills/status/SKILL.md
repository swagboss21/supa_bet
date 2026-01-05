---
name: status
description: Morning briefing with overnight results, cron health, today's games, and performance trends. Use when user says "status", "daily report", "morning briefing", or "what's going on".
allowed-tools: mcp__supabase__execute_sql
---

# Daily Status Report

Provides a comprehensive morning briefing showing system health, overnight results, today's schedule, pending bets, and performance trends.

## What This Skill Does

1. Checks cron job health (last 24 hours)
2. Shows overnight betting results (yesterday's settled bets)
3. Lists today's game schedule by sport
4. Displays any pending unsettled bets
5. Summarizes performance trends (rolling 7 days)

## Process

### Step 1: Cron Health Check

```sql
SELECT
  jobname as job,
  start_time AT TIME ZONE 'UTC' as last_run_utc,
  status,
  ROUND(EXTRACT(EPOCH FROM (end_time - start_time))::numeric, 1) as duration_sec
FROM cron.job_run_details
WHERE start_time > NOW() - INTERVAL '24 hours'
ORDER BY start_time DESC
LIMIT 20;
```

Alert if any jobs have status != 'succeeded' or duration > 30 seconds.

### Step 2: Overnight Results (Yesterday's Settled Bets)

```sql
SELECT
  sport,
  away_team || ' @ ' || home_team as game,
  selection as pick,
  result,
  final_score as score,
  odds
FROM prediction_analysis_view
WHERE DATE(settled_at AT TIME ZONE 'America/New_York') =
      (CURRENT_DATE AT TIME ZONE 'America/New_York' - INTERVAL '1 day')::date
  AND decision = 'BET'
ORDER BY settled_at;
```

Calculate W-L record and win percentage from the results.

### Step 3: Today's Schedule

```sql
SELECT
  sport,
  COUNT(*) as games,
  TO_CHAR(MIN(game_time) AT TIME ZONE 'America/New_York', 'HH:MI PM') as first_game_et
FROM game_analysis_view
WHERE game_date = CURRENT_DATE
GROUP BY sport
ORDER BY MIN(game_time);
```

Show total game count across all sports.

### Step 4: Pending Bets

```sql
SELECT
  sport,
  away_team || ' @ ' || home_team as game,
  game_date,
  selection,
  ROUND(max_confidence::numeric, 0) as confidence
FROM prediction_analysis_view
WHERE decision = 'BET'
  AND result IS NULL
  AND selection IS NOT NULL
ORDER BY game_date, game_time;
```

Alert if any bets are > 24 hours old and still pending.

### Step 5: Performance Trends (Rolling 7 Days)

```sql
SELECT
  CASE
    WHEN max_confidence >= 70 THEN '70%+'
    WHEN max_confidence >= 60 THEN '60-69%'
    ELSE '<60%'
  END as confidence_tier,
  COUNT(*) FILTER (WHERE result = 'WIN') as wins,
  COUNT(*) FILTER (WHERE result = 'LOSS') as losses,
  ROUND(100.0 * COUNT(*) FILTER (WHERE result = 'WIN') /
    NULLIF(COUNT(*) FILTER (WHERE result IN ('WIN', 'LOSS')), 0), 1) as win_pct
FROM prediction_analysis_view
WHERE decision = 'BET'
  AND result IS NOT NULL
  AND settled_at > NOW() - INTERVAL '7 days'
GROUP BY confidence_tier
ORDER BY confidence_tier DESC;
```

Also run overall stats:

```sql
SELECT
  COUNT(*) FILTER (WHERE result = 'WIN') as total_wins,
  COUNT(*) FILTER (WHERE result = 'LOSS') as total_losses,
  ROUND(100.0 * COUNT(*) FILTER (WHERE result = 'WIN') /
    NULLIF(COUNT(*) FILTER (WHERE result IN ('WIN', 'LOSS')), 0), 1) as overall_win_pct
FROM prediction_analysis_view
WHERE decision = 'BET'
  AND result IS NOT NULL
  AND settled_at > NOW() - INTERVAL '7 days';
```

Compare to 52.4% breakeven threshold.

## Output

Present results in this format:

```markdown
# Daily Status Report (2026-01-05 9:00 AM ET)

## Cron Health
| Job | Last Run (UTC) | Status | Duration |
|-----|----------------|--------|----------|
| refresh-gameday | 08:30 | succeeded | 2.1s |
| refresh-24hr | 08:00 | succeeded | 3.4s |
| settle-games | 06:00 | succeeded | 1.8s |

All systems operational.

## Overnight Results (2026-01-04)
| Sport | Game | Pick | Result | Score |
|-------|------|------|--------|-------|
| NBA | BKN @ CLE | CLE -5.5 | WIN | 98-112 |
| NHL | NYR @ BOS | BOS ML | LOSS | 2-3 |

**Yesterday:** 4W-3L (57.1%)

## Today's Schedule
| Sport | Games | First Game (ET) |
|-------|-------|-----------------|
| NCAAB | 8 | 2:00 PM |
| NBA | 5 | 7:00 PM |
| NHL | 4 | 7:00 PM |

**Total:** 17 games

## Pending Bets
| Game | Pick | Confidence |
|------|------|------------|
| LAL @ GSW | GSW -3.5 | 72% |

1 pending bet(s)

## Performance (Last 7 Days)
| Confidence | Record | Win % |
|------------|--------|-------|
| 70%+ | 3-1 | 75.0% |
| 60-69% | 8-6 | 57.1% |
| <60% | 2-2 | 50.0% |

**7-Day:** 13W-9L (59.1%) | Breakeven: 52.4%
```

## Important Notes

1. **Timezone handling** - Cron times are UTC, game times display in ET
2. **Uses existing views** - No new tables or functions required
3. **Rolling 7-day window** - Performance trends reflect recent accuracy
4. **Breakeven threshold** - Standard -110 juice requires 52.4% to profit
