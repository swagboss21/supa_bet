# Injury & Lineup Data Sources

**Research Date:** 2026-01-02 (Updated)
**Purpose:** Document all viable data sources for game-specific injuries and lineups
**Status:** Data inventory complete - ready for implementation design

---

## Implementation Status

| Feature | Documented | Implemented | Status |
|---------|------------|-------------|--------|
| ESPN JSON injuries | Yes | **NO** | `game_injuries` table empty (0 rows) |
| DailyFaceoff goalies | Yes | **NO** | No `game_goalies` table exists |
| DailyFaceoff lineups | Yes | **NO** | No `game_lineups` table exists |
| Injury cron | Yes | **NO** | `refresh-injuries` Edge Function deprecated |
| WebSearch fallback | Yes | **YES** | `game-researcher` agent fetches per-game |

**Current State:** All injury/lineup data is fetched inline by `game-researcher` agent via WebSearch. No automated cron populates structured tables. The `game_injuries` table exists but is unused.

---

## Executive Summary

| Category | Best Source | Format | Cron Viable? |
|----------|-------------|--------|--------------|
| **NBA Injuries** | ESPN JSON API | JSON | **YES** |
| **NHL Injuries** | ESPN JSON API | JSON | **YES** |
| **NFL Injuries** | ESPN JSON API | JSON | **YES** |
| **NCAAB Injuries** | N/A | Empty | No data available |
| **NHL Goalies** | DailyFaceoff | JSON in `__NEXT_DATA__` | **YES** |
| **NHL Line Combos** | DailyFaceoff Team Pages | JSON in `__NEXT_DATA__` | **YES** |
| **NBA Lineups** | N/A | JS-rendered | No viable source |
| **NFL Lineups** | N/A | 404/JS | No viable source |

**Key Discovery:** ESPN has **JSON injury APIs** that work (not the broken HTML pages).

---

## Source 1: ESPN JSON Injury APIs (NEW - RECOMMENDED)

### Status: CRON VIABLE - ALL SPORTS

**URLs:**
```
NBA:   https://site.api.espn.com/apis/site/v2/sports/basketball/nba/injuries
NHL:   https://site.api.espn.com/apis/site/v2/sports/hockey/nhl/injuries
NFL:   https://site.api.espn.com/apis/site/v2/sports/football/nfl/injuries
NCAAB: https://site.api.espn.com/apis/site/v2/sports/basketball/mens-college-basketball/injuries
       ↳ Returns empty array - no NCAAB injury data
```

### JSON Schema

```json
{
  "id": "511384",
  "longComment": "Young (quadriceps) won't play Friday against the Knicks...",
  "shortComment": "Missing third consecutive game",
  "status": "Out",
  "date": "2026-01-02T17:51Z",
  "athlete": {
    "firstName": "Trae",
    "lastName": "Young",
    "displayName": "Trae Young",
    "shortName": "T. Young",
    "position": {
      "id": "1",
      "name": "Point Guard",
      "displayName": "Point Guard",
      "abbreviation": "PG"
    },
    "team": {
      "id": "1",
      "displayName": "Atlanta Hawks",
      "abbreviation": "ATL"
    },
    "headshot": {
      "href": "https://a.espncdn.com/..."
    }
  },
  "details": {
    "fantasyStatus": {
      "description": "Out",
      "abbreviation": "O"
    },
    "type": "Quadriceps",
    "location": "Leg",
    "detail": "Bruise",
    "side": "Right",
    "returnDate": "2026-01-05"
  }
}
```

### Available Fields

| Field | Type | Example | Notes |
|-------|------|---------|-------|
| `id` | string | "511384" | Unique injury ID |
| `status` | string | "Out", "Day-To-Day", "Injured Reserve" | Current status |
| `longComment` | string | Full injury narrative | Detailed description |
| `shortComment` | string | Brief summary | One-liner |
| `date` | ISO 8601 | "2026-01-02T17:51Z" | Last updated |
| `athlete.displayName` | string | "Trae Young" | Player name |
| `athlete.position.abbreviation` | string | "PG", "C", "RW" | Position |
| `athlete.team.abbreviation` | string | "ATL", "BOS" | Team code |
| `details.type` | string | "Quadriceps", "Shoulder", "Upper Body" | Injury type |
| `details.location` | string | "Leg", "Arm", "Torso" | Body area |
| `details.detail` | string | "Bruise", "Surgery", "Sprain" | Specific injury |
| `details.side` | string | "Right", "Left", "Not Specified" | Which side |
| `details.returnDate` | date | "2026-02-25" | Expected return |

### Rate Limits & Authentication

| Aspect | Value |
|--------|-------|
| Authentication | **None required** (public API) |
| Rate limits | **None observed** |
| CORS | Allowed |
| Response time | <1s |
| Data freshness | Updates throughout day |

**Verified:** 2026-01-02 - API returns full injury list with no auth headers.

### Code Example

```typescript
const SPORT_PATHS = {
  NBA: 'basketball/nba',
  NHL: 'hockey/nhl',
  NFL: 'football/nfl',
  NCAAB: 'basketball/mens-college-basketball' // returns empty
};

async function fetchInjuries(sport: string): Promise<Injury[]> {
  const path = SPORT_PATHS[sport];
  const url = `https://site.api.espn.com/apis/site/v2/sports/${path}/injuries`;
  const res = await fetch(url);
  const data = await res.json();
  return data.items || [];
}
```

### Sample Data (NBA - 2026-01-02)

```
Atlanta Hawks:
- Trae Young | PG | Quadriceps Bruise (Right) | OUT
- N'Faly Dante | C | ACL Tear (Right Knee) | OUT (season-ending)

Boston Celtics:
- Jayson Tatum | F | Achilles Surgery (Right) | OUT (return April 2026)

Brooklyn Nets:
- Terance Mann | G | Hip Bruise (Right) | DAY-TO-DAY
- Nic Claxton | C | Personal | OUT
- Michael Porter Jr. | F | Illness | OUT
- Cam Thomas | G | Rest | OUT
- Haywood Highsmith | F | Knee Meniscus Surgery | OUT

Charlotte Hornets:
- Ryan Kalkbrenner | C | Elbow Sprain (Left) | OUT
- Miles Bridges | F | Ankle Sprain (Right) | DAY-TO-DAY
- Moussa Diabate | F | Wrist Sprain (Right) | DAY-TO-DAY
- Mason Plumlee | C | Groin Surgery (Right) | OUT
```

### Sample Data (NHL - 2026-01-02)

```
Anaheim Ducks:
- Frank Vatrano | RW | Shoulder | IR (return Feb 25)

Boston Bruins:
- Tanner Jeannot | LW | Undisclosed | Day-To-Day
- Henri Jokiharju | D | Undisclosed | IR

Buffalo Sabres:
- Michael Kesselring | D | Lower Body | Day-To-Day
- Tyson Kozak | C | Upper Body | Day-To-Day
- Alex Lyon | G | Lower Body | Out

Carolina Hurricanes:
- Shayne Gostisbehere | D | Groin | Day-To-Day
- Seth Jarvis | C | Upper Body | IR
- Pyotr Kochetkov | G | Hip | IR (season-ending)

Chicago Blackhawks:
- Frank Nazar | C | Upper Body | IR
```

### Sample Data (NFL - 2026-01-02)

```
Arizona Cardinals:
- Marvin Harrison Jr. | WR | Foot | IR
- Budda Baker | S | Concussion/Thumb | Active (cleared)
- Josh Sweat | LB | Ankle/Knee | Questionable
- Walter Nolen III | DT | Knee | IR (surgery)
- Kei'Trel Clark | CB | Back | IR (season-ending)
```

---

## Source 2: DailyFaceoff Starting Goalies

### Status: CRON VIABLE - NHL ONLY

**URL:** `https://www.dailyfaceoff.com/starting-goalies/`

**Data Access:** JSON embedded in `<script id="__NEXT_DATA__">` tag

### Sample Data (2026-01-02)

```
Game 1: Vegas Golden Knights @ St. Louis Blues
├── Home: Joel Hofer (Confirmed) - 8-8-2, 2.87 GAA, .900 SV%
├── Away: Carter Hart (Confirmed) - 4-2-3, 3.18 GAA, .881 SV%
├── Time: 2026-01-02T20:00:00.000Z
└── Spread: STL +1.5

Game 2: New York Rangers @ Florida Panthers (Winter Classic)
├── Home: Sergei Bobrovsky (Confirmed) - 17-9-1, 2.79 GAA, .887 SV%
├── Away: Igor Shesterkin (Unconfirmed) - 16-12-4, 2.51 GAA, .909 SV%
├── Time: 2026-01-03T01:00:00.000Z
└── Spread: FLA -1.5

Game 3: Minnesota Wild @ Anaheim Ducks
├── Home: Lukas Dostal (Confirmed) - 13-9-2, 3.15 GAA, .888 SV%
├── Away: Filip Gustavsson (Confirmed) - 13-8-4, 2.47 GAA, .912 SV%
├── Time: 2026-01-03T03:30:00.000Z
└── Spread: MIN -1.5

Game 4: Seattle Kraken @ Vancouver Canucks
├── Home: Thatcher Demko (Confirmed) - 8-8-0, 2.71 GAA, .904 SV%
├── Away: Joey Daccord (Unconfirmed) - 10-9-5, 2.77 GAA, .903 SV%
├── Time: 2026-01-03T03:30:00.000Z
└── Spread: VAN -1.5
```

### Available Fields

| Field | Type | Example |
|-------|------|---------|
| `{home\|away}TeamName` | string | "St Louis Blues" |
| `{home\|away}GoalieName` | string | "Joel Hofer" |
| `{home\|away}GoalieWins` | int | 8 |
| `{home\|away}GoalieLosses` | int | 8 |
| `{home\|away}GoalieOvertimeLosses` | int | 2 |
| `{home\|away}GoalieShutouts` | int | 3 |
| `{home\|away}GoalieSavePercentage` | string | "0.9" |
| `{home\|away}GoalieGoalsAgainstAvg` | string | "2.87" |
| `{home\|away}GoalieRating` | float | 25.0 (0-100 scale) |
| `{home\|away}NewsStrengthName` | string | "Confirmed", "Probable", "Unconfirmed" |
| `{home\|away}NewsStrengthId` | int | 2 = Confirmed |
| `{home\|away}NewsDetails` | string | "Hofer will start vs. Vegas on Friday." |
| `{home\|away}NewsCreatedAt` | ISO 8601 | "2026-01-02T18:50:21.405Z" |
| `{home\|away}NewsSourceName` | string | "Lou Korac" |
| `date` | date | "2026-01-02" |
| `time` | string | "15:00" (local) |
| `dateGmt` | ISO 8601 | "2026-01-02T20:00:00.000Z" |
| `pointSpread` | float | 1.5 |

### Extraction Method

```javascript
const response = await fetch('https://www.dailyfaceoff.com/starting-goalies/');
const html = await response.text();
const match = html.match(/<script id="__NEXT_DATA__" type="application\/json">(.*?)<\/script>/s);
const data = JSON.parse(match[1]);
const games = data.props.pageProps.data;
```

---

## Source 3: DailyFaceoff Team Line Combinations

### Status: CRON VIABLE - NHL ONLY

**URL Pattern:** `https://www.dailyfaceoff.com/teams/{team-slug}/line-combinations`

**Example:** `https://www.dailyfaceoff.com/teams/vegas-golden-knights/line-combinations`

**Data Access:** JSON embedded in `<script id="__NEXT_DATA__">` tag

### Sample Data (Vegas Golden Knights - 2026-01-02)

```
FORWARDS:
├── Line 1: Barbashev - Eichel - Bowman
├── Line 2: Howden - Marner - Stone
├── Line 3: Smith - Hertl - Dorofeyev
└── Line 4: Saad - Sissons - Kolesar

DEFENSE:
├── Pair 1: Hanifin - Whitecloud
├── Pair 2: Lauzon - Korczak
└── Pair 3: Megna - Hutton

POWER PLAY:
├── PP1: Dorofeyev, Hertl, Stone, Eichel, Marner
└── PP2: Barbashev, Sissons, Bowman, Holtz, Hanifin

PENALTY KILL:
├── PK1: Howden, Stone, Hanifin, Whitecloud
└── PK2: Sissons, Marner, Korczak, Hutton

GOALIES:
├── Starter: Carter Hart (#79)
└── Backup: Akira Schmid (#40)

INJURED RESERVE:
├── Alex Pietrangelo (#7) - IR
├── Adin Hill (#33) - IR
├── William Karlsson (#71) - IR
└── Shea Theodore (#27) - Day-to-Day
```

### Sample Data (Boston Bruins - 2026-01-02)

```
FORWARDS:
├── Line 1: Steeves - Lindholm - Geekie
├── Line 2: Khusnutdinov - Minten - Pastrnak
├── Line 3: Mittelstadt - Zacha - Arvidsson
└── Line 4: Eyssimont - Kuraly - Kastelic

DEFENSE:
├── Pair 1: Zadorov - McAvoy
├── Pair 2: Lindholm - Aspirot
└── Pair 3: Lohrei - Peeke

GOALIES:
├── Starter: Jeremy Swayman
└── Backup: Joonas Korpisalo

INJURED RESERVE:
├── Jordan Harris - Ankle (surgery) - IR
├── Henri Jokiharju - Undisclosed - IR
├── Tanner Jeannot - Undisclosed - Day-to-Day
├── Matej Blumel - Lower body - IR (assigned to AHL)
└── Michael Callahan - Undisclosed - IR (assigned to AHL)
```

### Available Fields

| Category | Fields |
|----------|--------|
| **Forward Lines** | LW, C, RW for lines 1-4 |
| **Defense Pairs** | LD, RD for pairs 1-3 |
| **Power Play** | 5 players per unit (PP1, PP2) |
| **Penalty Kill** | 4 players per unit (PK1, PK2) |
| **Goalies** | Starter, Backup |
| **Injured Reserve** | Player, Injury type, Status |

### All 32 NHL Teams Available

URL pattern: `/teams/{team-slug}/line-combinations`

Teams: anaheim-ducks, boston-bruins, buffalo-sabres, calgary-flames, carolina-hurricanes, chicago-blackhawks, colorado-avalanche, columbus-blue-jackets, dallas-stars, detroit-red-wings, edmonton-oilers, florida-panthers, los-angeles-kings, minnesota-wild, montreal-canadiens, nashville-predators, new-jersey-devils, new-york-islanders, new-york-rangers, ottawa-senators, philadelphia-flyers, pittsburgh-penguins, san-jose-sharks, seattle-kraken, st-louis-blues, tampa-bay-lightning, toronto-maple-leafs, utah-mammoth, vancouver-canucks, vegas-golden-knights, washington-capitals, winnipeg-jets

---

## Source 4: ESPN HTML Injury Pages

### Status: NOT VIABLE (Client-Side Rendered)

**URLs:**
- NBA: `https://www.espn.com/nba/injuries`
- NHL: `https://www.espn.com/nhl/injuries`
- NFL: `https://www.espn.com/nfl/injuries`

**Why Broken:** React client-side rendering. Initial HTML is empty shell.

**Note:** Use JSON API instead (Source 1).

---

## Source 5: RotoWire

### Status: NOT VIABLE (JavaScript Rendered)

**URLs:**
- NBA: `https://www.rotowire.com/basketball/nba-lineups.php`
- NHL: `https://www.rotowire.com/hockey/nhl-lineups.php`
- NFL: `https://www.rotowire.com/football/nfl-lineups.php` (404)
- Injury Report: `https://www.rotowire.com/basketball/injury-report.php`

**Why Not Viable:** Pages return only navigation/script tags. Actual content is JavaScript-rendered. Would require Playwright/Puppeteer.

---

## Source 6: CBS Sports

### Status: NOT VIABLE (JavaScript Rendered)

**URLs:**
- NBA: `https://www.cbssports.com/nba/injuries/`
- NHL: `https://www.cbssports.com/nhl/injuries/`

**Why Not Viable:** Same as RotoWire - JavaScript client-side rendering.

---

## Source 7: DailyFaceoff Injuries Page

### Status: NOT AVAILABLE

**URL:** `https://www.dailyfaceoff.com/injuries/`

**Result:** 404 Not Found

**Alternative:** Injury data available per-team via line combinations pages (Source 3).

---

## Dead/Unavailable Sources

| Source | URL | Status |
|--------|-----|--------|
| DailyFaceoff /injuries | dailyfaceoff.com/injuries/ | 404 |
| RotoWire NBA injuries | rotowire.com/basketball/injuries.php | 404 |
| RotoWire NFL Lineups | rotowire.com/football/nfl-lineups.php | 404 |
| Covers.com injuries | covers.com/sport/injuries | 404 |
| CBS Sports | cbssports.com/nba/injuries | JS-rendered |
| ESPN NCAAB injuries | JSON API returns empty array | No data |

---

## Data Requirements vs Availability

### What We Need for Predictions

| Data | Sport | Needed For | Priority |
|------|-------|------------|----------|
| Injury status | ALL | Impact assessment | P0 |
| Expected return | ALL | Game availability | P0 |
| Starting goalie | NHL | Goalie-based betting | P0 |
| Goalie stats | NHL | SV%, GAA for matchup | P1 |
| Confirmation status | NHL | Bet timing | P1 |
| Line combinations | NHL | PP/PK analysis | P2 |
| Starting lineup | NBA | Minutes distribution | P2 |

### What's Available via Cron

| Data | Source | Sport Coverage | Quality |
|------|--------|----------------|---------|
| Injury status | ESPN JSON | NBA, NHL, NFL | Complete |
| Expected return | ESPN JSON | NBA, NHL, NFL | When available |
| Injury type | ESPN JSON | NBA, NHL, NFL | Detailed |
| Starting goalie | DailyFaceoff | NHL | Excellent |
| Goalie stats | DailyFaceoff | NHL | Complete |
| Confirmation status | DailyFaceoff | NHL | Real-time |
| Line combinations | DailyFaceoff | NHL | Complete |
| IR list | DailyFaceoff | NHL | Per-team |
| Starting lineup | N/A | NBA, NFL | **NOT AVAILABLE** |

### Gaps

| Gap | Sport | Workaround |
|-----|-------|------------|
| Starting lineups | NBA | WebSearch per-game |
| Starting lineups | NFL | WebSearch per-game |
| NCAAB injuries | NCAAB | WebSearch per-game |
| Late scratches | ALL | WebSearch per-game (agent) |
| GTD updates | ALL | WebSearch per-game (agent) |

---

## Cron Implementation Summary

### Can Automate (No Claude Required)

```
ESPN JSON API (injuries):
├── NBA injuries → game_injuries table
├── NHL injuries → game_injuries table
├── NFL injuries → game_injuries table
└── Schedule: Every 2hr, more frequent on gameday

DailyFaceoff (NHL only):
├── Starting goalies → game_goalies table
├── Goalie stats → game_goalies table
├── Confirmation status → game_goalies table
├── Line combinations → game_lineups table
├── IR list → game_injuries table (supplement)
└── Schedule: Every 2hr on gameday (after morning skate)
```

### Requires Claude Agent

```
Per-game WebSearch:
├── NBA starting lineups (30 min before tipoff)
├── NFL starting lineups (90 min before kickoff)
├── NCAAB injuries (no structured source)
├── Late scratches (< 1hr before game)
└── GTD status changes (news interpretation)
```

---

## Data Freshness Requirements

For betting, timing matters. Different data types have different freshness requirements:

| Data Type | Minimum Freshness | Ideal Timing | Why |
|-----------|-------------------|--------------|-----|
| **NHL Goalie** | <2 hours | After morning skate (~10am local) | Goalies often announced after practice |
| **NBA Injuries** | <4 hours | 90 min pre-game | GTD decisions made close to tip |
| **NFL Injuries** | <6 hours | Friday injury report | Final designations on Friday |
| **Late Scratches** | <30 min | Pre-game | Can move lines significantly |
| **Lineup Changes** | <1 hour | Pre-game | Rest days, rotations |

### Goalie Announcement Timeline (NHL)

```
Morning (10am-12pm ET): Most goalies announced after morning skate
Afternoon (2pm-5pm): Remaining goalies typically confirmed
Evening (6pm-7pm): Last-minute changes rare but possible
```

**DailyFaceoff Confirmation Levels:**
- `Confirmed` (strength 2): Official announcement
- `Probable` (strength 1): Strong indication
- `Unconfirmed` (strength 0): Projection only

**Betting Recommendation:** Only bet NHL totals/goalies after "Confirmed" status.

---

## API Response Sizes (Token Estimates)

| Source | Endpoint | Response Size | Fetch Time |
|--------|----------|---------------|------------|
| ESPN NBA injuries | /injuries | ~50KB | <1s |
| ESPN NHL injuries | /injuries | ~40KB | <1s |
| ESPN NFL injuries | /injuries | ~60KB | <1s |
| DailyFaceoff goalies | /starting-goalies/ | ~100KB HTML | <2s |
| DailyFaceoff team | /teams/{slug}/line-combinations | ~80KB HTML | <2s |

---

## Alternative Paid APIs (For Reference)

If free sources become unreliable, these paid APIs offer comprehensive injury data:

| Provider | Sports | Free Tier | Injury Data | URL |
|----------|--------|-----------|-------------|-----|
| **BallDontLie** | NBA, NHL, NFL, MLB, NCAAB | 5 req/min | Yes | [balldontlie.io](https://www.balldontlie.io/) |
| **MySportsFeeds** | NBA, NHL, NFL, MLB | Personal use free | Yes | [mysportsfeeds.com](https://www.mysportsfeeds.com/) |
| **SportsDataIO** | All major | Trial only | Yes | [sportsdata.io](https://sportsdata.io/) |
| **Sportradar** | All major | None | Yes | [sportradar.com](https://developer.sportradar.com/) |
| **API-Sports** | All major | Limited | Yes | [api-sports.io](https://api-sports.io/) |

### ESPN Team-Specific Injury Endpoints

Undocumented but working team-level endpoints:

```
https://sports.core.api.espn.com/v2/sports/football/leagues/nfl/teams/{TEAM_ID}/injuries
https://sports.core.api.espn.com/v2/sports/basketball/leagues/nba/teams/{TEAM_ID}/injuries
https://sports.core.api.espn.com/v2/sports/hockey/leagues/nhl/teams/{TEAM_ID}/injuries
```

Team IDs can be found in scoreboard responses.

---

## Team Name Mapping

**Problem:** Different sources use different team name formats.

| Team | ESPN API | DailyFaceoff | Action Network |
|------|----------|--------------|----------------|
| St. Louis Blues | `STL` | `St Louis Blues` | `St. Louis` |
| Vegas Golden Knights | `VGK` | `Vegas Golden Knights` | `Vegas` |
| New York Rangers | `NYR` | `New York Rangers` | `NY Rangers` |
| Los Angeles Kings | `LA` | `Los Angeles Kings` | `LA Kings` |
| Tampa Bay Lightning | `TB` | `Tampa Bay Lightning` | `Tampa Bay` |

**Recommendation:** Store canonical names in `games` table, create lookup functions for each source.

### DailyFaceoff Team Slugs (32 NHL Teams)

```
anaheim-ducks, boston-bruins, buffalo-sabres, calgary-flames,
carolina-hurricanes, chicago-blackhawks, colorado-avalanche, columbus-blue-jackets,
dallas-stars, detroit-red-wings, edmonton-oilers, florida-panthers,
los-angeles-kings, minnesota-wild, montreal-canadiens, nashville-predators,
new-jersey-devils, new-york-islanders, new-york-rangers, ottawa-senators,
philadelphia-flyers, pittsburgh-penguins, san-jose-sharks, seattle-kraken,
st-louis-blues, tampa-bay-lightning, toronto-maple-leafs, utah-hockey-club,
vancouver-canucks, vegas-golden-knights, washington-capitals, winnipeg-jets
```

**Note:** Arizona Coyotes relocated to Utah Hockey Club for 2024-25 season.

---

## WebSearch Fallback Queries

When structured APIs fail, use these WebSearch patterns (from `game-researcher` agent):

### Injuries

```
"{away_team} vs {home_team} injuries {game_date}"
"{team_name} injury report today"
"{team_name} {player_name} injury status"
```

### Lineups (NBA)

```
"{away_team} vs {home_team} starting lineup {game_date}"
"{team_name} starting five tonight"
```

### Goalies (NHL)

```
"{home_team} starting goalie tonight"
"{away_team} starting goalie tonight"
"{team_name} goalie confirmed"
```

### Late Scratches

```
"{away_team} vs {home_team} scratches"
"{team_name} late scratch {game_date}"
```

---

## Actual Cron Schedule (Database)

Current pg_cron jobs (as of 2026-01-02):

| Job | Schedule | Edge Function | Tier |
|-----|----------|---------------|------|
| `refresh-opening-lines` | Daily 9am UTC | `refresh-odds` | opening |
| `refresh-72hr` | Every 6 hours | `refresh-odds` | 72hr |
| `refresh-24hr` | Every 2 hours | `refresh-odds` | 24hr |
| `refresh-gameday` | Every 30 min (game hours) | `refresh-odds` | gameday |
| `settle-games` | 6am, 8am, 10am UTC | `settle-games` | - |

**Note:** No injury refresh cron exists. The `refresh-injuries` Edge Function is deployed but not scheduled.

---

## Error Handling Recommendations

| Scenario | Fallback |
|----------|----------|
| ESPN JSON 5xx | Retry 3x with exponential backoff |
| ESPN JSON empty | Log warning, continue without injuries |
| DailyFaceoff 403 | Switch to WebSearch for goalie |
| DailyFaceoff parse fail | Check if `__NEXT_DATA__` structure changed |
| All sources fail | Use stale data from `game_injuries` if <24hr old |

---

## Future Work

### P0 - Enable Automated Injury Refresh

1. Create new Edge Function using ESPN JSON API (not HTML)
2. Add pg_cron job: Every 2hr on gameday
3. Match injuries to upcoming games via team abbreviation

### P1 - Add NHL Goalie Table

1. Create `game_goalies` table
2. Edge Function to fetch DailyFaceoff `__NEXT_DATA__`
3. Add pg_cron job: Every 2hr on NHL gamedays (after morning skate)

### P2 - NCAAB Injuries

1. No structured source exists
2. Option A: Paid API (BallDontLie)
3. Option B: WebSearch per-game (current approach)

---

*This document is data inventory only. Implementation design is separate.*
