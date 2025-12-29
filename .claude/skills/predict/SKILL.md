---
name: predict
description: Analyze today's games and make betting picks. Use when user says "predict games", "analyze today's slate", "make picks", or "find edges".
allowed-tools: Read, WebSearch, WebFetch
---

# Predict Games

Analyze today's games from Supabase and make betting picks based on public betting data, sharp action, and injury context.

## Prerequisites

Run `/refresh-games` first to ensure Supabase has today's games with injuries.

## What This Skill Does

1. Queries Supabase for today's games (already includes odds and injuries)
2. For each sport, delegates to sport-specific researcher agent
3. Each researcher searches for public betting % and social sentiment
4. Orchestrator verifies predictions and applies confidence thresholds
5. Qualifying picks are inserted into `placed_bets` table

## Workflow

```
/predict
    │
    ├── Query Supabase (games + injuries)
    │
    ├── NBA games → nba-researcher agent
    ├── NHL games → nhl-researcher agent
    ├── NFL games → nfl-researcher agent
    └── NCAAB games → ncaab-researcher agent
         │
         ├── public-betting skill (search Action Network, Covers)
         └── social-sentiment skill (search X, Reddit)
              │
              └── Return structured predictions
                   │
                   └── Orchestrator verifies → Insert to placed_bets
```

## Pass Criteria (v1)

- **PASS** if conflicting signals between sources
- **PASS** if confidence < 60%
- **PASS** if public betting data unavailable
- **BET** if clear public fade opportunity (>65% public on one side with sharp action opposite)

## Output

```
## Today's Picks (2025-12-29)

### NBA
| Game | Pick | Type | Odds | Confidence | Reasoning |
|------|------|------|------|------------|-----------|
| HOU @ MIA | Heat +3.5 | Spread | -110 | 72% | 68% public on Rockets, sharps on Heat |

### NCAAB
| Game | Pick | Type | Odds | Confidence | Reasoning |
|------|------|------|------|------------|-----------|
| Duke @ Wake | Wake +7.5 | Spread | -110 | 65% | Heavy public on Duke |

### Passed Games
- BOS @ CLE: No clear edge (public split 52/48)
- LAL @ PHX: Conflicting signals

Total: 2 picks placed, 28 games passed
```
