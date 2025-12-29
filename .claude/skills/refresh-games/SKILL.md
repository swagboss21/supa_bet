---
name: refresh-games
description: Refresh today's games, odds, and injuries from ESPN. Use when user says "refresh games", "update slate", "pull today's games", or "get today's odds".
allowed-tools: WebFetch, WebSearch
---

# Refresh Games

Scrape ESPN for today's games, odds, and injuries across all sports (NBA, NHL, NFL, NCAAB).

## What This Skill Does

1. Fetches today's odds from ESPN for each sport
2. Fetches injury reports from ESPN for each sport
3. Parses the data into structured format
4. Updates Supabase `games` table with odds + injuries

## ESPN URLs to Scrape

### Odds Pages
- NBA: `https://www.espn.com/nba/odds`
- NHL: `https://www.espn.com/nhl/odds`
- NFL: `https://www.espn.com/nfl/odds`
- NCAAB: `https://www.espn.com/mens-college-basketball/odds`

### Injury Pages
- NBA: `https://www.espn.com/nba/injuries`
- NHL: `https://www.espn.com/nhl/injuries`
- NFL: `https://www.espn.com/nfl/injuries`
- NCAAB: `https://www.espn.com/mens-college-basketball/injuries`

## Data Format

For each game, extract:
- `sport`: NBA, NHL, NFL, or NCAAB
- `game_date`: Today's date
- `game_time`: Game start time
- `away_team`: Away team name
- `home_team`: Home team name
- `away_ml`: Away moneyline (integer, e.g., -155 or +130)
- `home_ml`: Home moneyline
- `spread`: Point spread (positive = home favored)
- `total`: Over/under
- `injuries`: JSONB with format `{"away": ["Player (Status)"], "home": ["Player (Status)"]}`

## Supabase Update

Use upsert logic to update existing games or insert new ones:

```sql
INSERT INTO games (sport, game_date, game_time, away_team, home_team, away_ml, home_ml, spread, total, injuries, book)
VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9, $10, 'ESPN/DraftKings')
ON CONFLICT (sport, game_date, away_team, home_team)
DO UPDATE SET
  away_ml = EXCLUDED.away_ml,
  home_ml = EXCLUDED.home_ml,
  spread = EXCLUDED.spread,
  total = EXCLUDED.total,
  injuries = EXCLUDED.injuries;
```

## Output

Return a summary of games refreshed:
```
Refreshed games for 2025-12-29:
- NBA: 11 games
- NHL: 11 games
- NFL: 1 game
- NCAAB: 7 games
Total: 30 games with odds and injuries
```
