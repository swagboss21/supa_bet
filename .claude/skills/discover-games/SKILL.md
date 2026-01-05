---
name: discover-games
description: Find upcoming games 3-7 days out to capture opening lines. Use when user says "find upcoming games", "discover new games", or "get next week's slate".
allowed-tools: WebFetch
---

# Discover Games

Find games 3-7 days in the future to capture opening lines before the scheduled refresh cycle.

## What This Skill Does

1. Fetches ESPN scoreboard for upcoming dates (3-7 days out)
2. Identifies new games not yet in Supabase
3. Creates game records with ESPN event IDs
4. Sets up refresh schedules for opening line capture

## Why This Matters

Opening lines are crucial for:
- Detecting line movement (sharp money indicators)
- Finding value before public betting moves lines
- Tracking reverse line movement (RLM)

Lines typically open 72+ hours before game time. This skill ensures we capture them.

## ESPN Scoreboard API

For each sport and date, fetch:
```
https://site.api.espn.com/apis/site/v2/sports/{sport}/{league}/scoreboard?dates=YYYYMMDD
```

| Sport | Path |
|-------|------|
| NBA | basketball/nba |
| NHL | hockey/nhl |
| NFL | football/nfl |
| NCAAB | basketball/mens-college-basketball |

## Process

### Step 1: Generate Date Range

Calculate dates 3-7 days from now:
```javascript
const dates = [];
for (let i = 3; i <= 7; i++) {
  const d = new Date();
  d.setDate(d.getDate() + i);
  dates.push(d.toISOString().slice(0, 10).replace(/-/g, ''));
}
// Results in: ['20250101', '20250102', '20250103', '20250104', '20250105']
```

### Step 2: Fetch Scoreboard for Each Sport/Date

Use WebFetch to get JSON from ESPN:
```
https://site.api.espn.com/apis/site/v2/sports/basketball/nba/scoreboard?dates=20250101
```

Extract from response:
- `events[].id` - ESPN event ID
- `events[].date` - Game time
- `events[].competitions[0].competitors` - Teams

### Step 3: Check for Existing Games

Query Supabase to avoid duplicates:
```sql
SELECT id, espn_event_id
FROM games
WHERE sport = 'NBA'
  AND game_date = '2025-01-01'
  AND (espn_event_id = '401810350' OR (away_team ILIKE '%Celtics%' AND home_team ILIKE '%Heat%'));
```

### Step 4: Insert New Games

For new games, insert with ESPN event ID:
```sql
-- game_time is TIMESTAMPTZ (UTC). Use ISO format with Z suffix.
INSERT INTO games (sport, game_date, game_time, away_team, home_team, espn_event_id, status)
VALUES ('NBA', '2025-01-01', '2025-01-01T19:30:00Z', 'Boston Celtics', 'Miami Heat', '401810350', 'scheduled')
ON CONFLICT (sport, game_date, away_team, home_team)
DO UPDATE SET espn_event_id = COALESCE(games.espn_event_id, EXCLUDED.espn_event_id)
RETURNING id;
```

## Automated Discovery

Discovery runs automatically as part of the `refresh-opening-lines` cron job:
- Runs daily at 9am UTC
- Looks 3-7 days ahead
- Creates games in Supabase with ESPN event IDs
- pg_cron handles refresh scheduling automatically (no manual scheduling needed)

## Manual Discovery

To trigger opening line discovery manually:
```sql
-- Trigger opening refresh via Edge Function
SELECT net.http_post(
  'https://htnhszioyydsjzkqeowx.supabase.co/functions/v1/refresh-odds',
  '{"tier": "opening"}'::jsonb,
  '{"Content-Type": "application/json"}'::jsonb
);
```

## Output

Return a summary:
```
## Games Discovered

| Sport | Date | New Games | Existing |
|-------|------|-----------|----------|
| NBA   | Jan 1 | 5 | 2 |
| NBA   | Jan 2 | 8 | 0 |
| NHL   | Jan 1 | 7 | 1 |
| NHL   | Jan 2 | 6 | 0 |
| NFL   | Jan 5 | 16 | 0 |
| NCAAB | Jan 1 | 12 | 3 |

**Total**: 54 new games discovered
**Next opening refresh**: 9am UTC tomorrow

### Sample Games Found
- Jan 1: Celtics @ Heat (NBA)
- Jan 1: Bruins @ Rangers (NHL)
- Jan 5: Cowboys @ Eagles (NFL)
```
