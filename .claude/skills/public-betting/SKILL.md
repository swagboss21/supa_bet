---
name: public-betting
description: Search for public betting percentages and money splits on games. Use when you need to find what percentage of bets are on each side of a game.
allowed-tools: WebSearch, WebFetch
---

# Public Betting Skill

Search the web for public betting percentages, money splits, and consensus picks.

## What This Skill Does

Searches reputable betting data sources to find:
- **Public bet %**: What percentage of bets are on each side
- **Money %**: What percentage of money is on each side (often different from bet %)
- **Line movement**: How the line has moved since opening
- **Consensus picks**: Expert/public pick summaries

## Data Sources to Search

### Primary Sources (Most Reliable)
1. **Action Network** - `actionnetwork.com`
   - Has bet % and money %
   - Shows sharp vs public splits
   - Tracks line movement

2. **Covers** - `covers.com/sports/[sport]/matchup`
   - Public consensus percentages
   - Line history
   - Expert picks

3. **VegasInsider** - `vegasinsider.com`
   - Betting percentages
   - Line movement
   - Consensus

### Secondary Sources
4. **ESPN Chalk** - betting coverage
5. **The Lines** - thelineups.com
6. **Odds Shark** - oddsshark.com

## Search Strategies

### For a specific game:
```
"[Team A] [Team B] public betting percentage"
"[Team A] vs [Team B] consensus picks"
"Action Network [Team A] [Team B] betting splits"
```

### For today's slate:
```
"[sport] public betting percentages today"
"[sport] betting consensus [date]"
"Action Network [sport] sharp vs public"
```

## Output Format

Return structured data for the game:

```json
{
  "game": "Milwaukee Bucks @ Charlotte Hornets",
  "sport": "NBA",
  "source": "Action Network",
  "data": {
    "spread": {
      "public_pct_away": 68,
      "public_pct_home": 32,
      "money_pct_away": 55,
      "money_pct_home": 45,
      "opening_line": -4.0,
      "current_line": -3.5,
      "line_movement": "toward home"
    },
    "moneyline": {
      "public_pct_away": 72,
      "public_pct_home": 28
    },
    "total": {
      "public_pct_over": 55,
      "public_pct_under": 45,
      "opening_total": 225.0,
      "current_total": 224.5
    }
  },
  "interpretation": "Heavy public on Bucks (68% spread, 72% ML), but money more balanced (55%). Line moved toward Hornets despite public action = potential sharp money on Charlotte."
}
```

## Key Signals to Identify

### Reverse Line Movement (RLM)
When the line moves OPPOSITE to public betting:
- Public on Team A (70%), but line moves toward Team B
- This suggests sharp money on Team B
- Strong fade signal

### Money vs Bets Discrepancy
- 70% of bets on Team A, but only 50% of money
- Suggests many small public bets vs fewer large sharp bets on Team B
- Worth investigating further

### Heavy Public (>65%)
- When public is heavily on one side
- Creates potential fade opportunity
- Especially strong with RLM or sharp money opposite

## Important Notes

- Not all games have public betting data available
- Data quality varies by source
- Always note the source of data
- If data unavailable, return `{"data": null, "reason": "No public data found"}`
- Prioritize Action Network and Covers for reliability
