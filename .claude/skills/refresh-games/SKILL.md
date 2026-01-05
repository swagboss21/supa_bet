---
name: refresh-games
description: Refresh today's games, odds, and injuries from ESPN. Use when user says "refresh games", "update slate", "pull today's games", or "get today's odds".
allowed-tools: WebFetch, WebSearch
---

# Refresh Games

Trigger the refresh-odds Edge Function or manually scrape ESPN JSON APIs for games, odds, and injuries.

## What This Skill Does

1. Calls the `refresh-odds` Edge Function (preferred)
2. Falls back to ESPN JSON API scraping if needed
3. Updates Supabase with games, odds, injuries, and line movement

## Primary Method: Edge Function

The system now runs automated scheduled refreshes via pg_cron. For manual override:

```bash
# Call the Edge Function directly
curl -X POST https://htnhszioyydsjzkqeowx.supabase.co/functions/v1/refresh-odds \
  -H "Content-Type: application/json" \
  -d '{"tier": "all", "force": true}'
```

Options:
- `tier`: "opening" | "72hr" | "24hr" | "6hr" | "gameday" | "all"
- `force`: true to skip schedule checks
- `sport`: "NBA" | "NHL" | "NFL" | "NCAAB" (optional, defaults to all)

## Automated Schedule (pg_cron)

The Edge Function runs automatically on this schedule:

| Job | Frequency | Purpose |
|-----|-----------|---------|
| refresh-opening-lines | Daily 9am UTC | Discover games 3-7 days out |
| refresh-72hr | Every 6 hours | Games 2-3 days out |
| refresh-24hr | Every 2 hours | Games tomorrow |
| refresh-gameday | Every 30 min | Today's games |
| settle-games | 8pm/10pm/midnight UTC | Settlement check |

## ESPN JSON APIs (Fallback)

If Edge Function fails, use these endpoints:

### Scoreboard (get event IDs)
```
https://site.api.espn.com/apis/site/v2/sports/{sport}/{league}/scoreboard?dates=YYYYMMDD
```

| Sport | Path |
|-------|------|
| NBA | basketball/nba |
| NHL | hockey/nhl |
| NFL | football/nfl |
| NCAAB | basketball/mens-college-basketball |

### Odds (per event)
```
https://sports.core.api.espn.com/v2/sports/{sport}/leagues/{league}/events/{EVENT_ID}/competitions/{EVENT_ID}/odds
```

Returns structured JSON with:
- Current spread, total, moneylines
- Opening lines for line movement tracking

### Injuries (JSON API - automated via Edge Function)
```
https://site.api.espn.com/apis/site/v2/sports/{sport}/{league}/injuries
```
- Fetched automatically by `refresh-odds` Edge Function
- Updates `game_injuries` and `injury_history` tables
- NCAAB returns empty (no data available from ESPN)

## Data Flow

```
ESPN JSON APIs → Edge Function → Supabase Tables
                                 ├── games (+ espn_event_id)
                                 ├── game_odds (current)
                                 ├── odds_history (snapshots)
                                 └── game_injuries
```

## Checking Cron Status

View cron job status:
```sql
SELECT * FROM cron_job_status;
```

View recent runs:
```sql
SELECT * FROM cron.job_run_details
ORDER BY start_time DESC
LIMIT 10;
```

## Output

Return a summary:
```
## Refresh Complete

**Method**: Edge Function (refresh-odds)
**Tier**: gameday

| Sport | Games | With Odds | Line Moves |
|-------|-------|-----------|------------|
| NBA   | 11    | 11        | 3          |
| NHL   | 11    | 11        | 2          |
| NFL   | 1     | 1         | 0          |
| NCAAB | 7     | 7         | 1          |

**Total**: 30 games refreshed
**Next scheduled refresh**: 30 minutes
```
