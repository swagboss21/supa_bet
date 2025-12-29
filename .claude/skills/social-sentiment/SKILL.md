---
name: social-sentiment
description: Search social media for sharp betting commentary and public sentiment. Use when you need to find what professional bettors and sharp money is doing.
allowed-tools: WebSearch, WebFetch
---

# Social Sentiment Skill

Search social media and betting communities for sharp action, professional handicapper takes, and public sentiment.

## What This Skill Does

Searches for:
- **Sharp bettor commentary**: What professional bettors are saying
- **Line movement analysis**: Why lines are moving
- **Handicapper picks**: Professional sports bettors' takes
- **Community sentiment**: Reddit, Twitter discussions
- **News that moves lines**: Information sharps are acting on

## Sources to Search

### Sharp/Professional Sources
1. **Twitter/X** - `site:twitter.com` or `site:x.com`
   - @TheSharpPlays
   - @CSSports_Action
   - @PFF_Bet (NFL)
   - @SportsInsights
   - Search for "[team] sharp money"

2. **Action Network Articles**
   - Sharp reports
   - "Betting market analysis"
   - Line movement breakdowns

### Community Sources
3. **Reddit**
   - r/sportsbook - daily picks threads
   - r/sportsbetting
   - r/nba, r/nfl, r/hockey for news
   - Search: `site:reddit.com/r/sportsbook [teams]`

4. **Discord/Forums** (via search)
   - Betting community discussions
   - Sharp group leaks

## Search Strategies

### For sharp action:
```
"[Team A] [Team B] sharp money"
"[Team A] sharp action today"
"[sport] sharp plays [date]"
"professional bettors [Team A]"
```

### For social sentiment:
```
site:reddit.com/r/sportsbook "[Team A]"
site:twitter.com "[Team A] [Team B] bet"
"[sport] best bets today sharp"
```

### For specific angles:
```
"[Team A] [Team B] weather forecast" (NFL outdoor games)
"[Team A] starting goalie confirmed" (NHL)
"[Player] status tonight" (if injury news needed)
```

## Output Format

Return findings in structured format:

```json
{
  "game": "Milwaukee Bucks @ Charlotte Hornets",
  "sport": "NBA",
  "findings": {
    "sharp_action": {
      "detected": true,
      "direction": "Charlotte Hornets",
      "sources": ["Action Network RLM report", "@TheSharpPlays tweet"],
      "confidence": "medium"
    },
    "social_sentiment": {
      "reddit_consensus": "mixed, slight lean public on Bucks",
      "twitter_notable": "@SharpBettor123 on Hornets at home",
      "overall": "public on Bucks, sharps potentially on Hornets"
    },
    "relevant_news": {
      "items": [
        "Giannis GTD - still being evaluated",
        "Hornets 4-1 ATS at home this month"
      ]
    }
  },
  "interpretation": "Sharp money appears to be on Hornets despite public on Bucks. Line hasn't moved much suggesting books want Bucks money. Potential fade spot if Giannis status clarifies."
}
```

## Key Things to Look For

### Sharp Money Indicators
- Professional bettors posting plays
- Steam moves (sudden line shifts)
- Reverse line movement discussions
- "Whale money" reports

### Contrarian Signals
- Overwhelming public sentiment one way
- Sharp voices going other way
- Line moving against public

### Red Flags
- Heavy public + no sharp counter = stay away
- Conflicting sharp opinions = uncertainty
- Late breaking news changing everything

## Sport-Specific Searches

### NBA
- Load management news
- Back-to-back schedules
- Star player rest

### NHL
- Starting goalie confirmations
- Backup goalie situations
- Travel schedules

### NFL
- Weather forecasts (outdoor games)
- Key injury practice reports
- Prime time game narratives

### NCAAB
- Blue blood fading opportunities
- Conference tournament situation
- Travel/scheduling disadvantages

## Important Notes

- Social media can have noise - focus on credible accounts
- Recent posts (last 24h) are most relevant
- Sharp money moves fast - timing matters
- If no clear signal found, say so: `{"sharp_action": {"detected": false}}`
- Don't fabricate sources - only report what you actually find
