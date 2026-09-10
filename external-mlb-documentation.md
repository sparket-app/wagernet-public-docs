# MLB Data Model in WagerNet

How Major League Baseball data maps into WagerNet.

See [General Documentation](external-general-documentation.md) for general WagerNet API documentation.

---

## Data Hierarchy

```
Discipline: baseball (URI: baseball)
└── Competition: MLB (URI: mlb)
    └── Season: mlb-2026
        ├── Stage: Spring Training (stage_type: spring-training)
        ├── Stage: Regular Season (stage_type: regular-season)
        ├── Stage: Wild Card Series (stage_type: wild-card)
        ├── Stage: Division Series (stage_type: division-series)
        ├── Stage: Championship Series (stage_type: championship-series)
        └── Stage: World Series (stage_type: world-series)
            └── Event (event_type=match)
```

- **Stage** = season phase. Stages are ordered by `stage_number` (1 = Spring Training through 6 = World Series).
- **Event** = individual game between two teams.

MLB events are flat (no parent-child nesting). Each game is a standalone event.

---

## Quick Start

```bash
# Upcoming baseball games
curl "https://bf3zb3ipuy.us-east-1.awsapprunner.com/api/v1/events?discipline_uri=baseball&status=upcoming&pageSize=100"

# Games in progress
curl "https://bf3zb3ipuy.us-east-1.awsapprunner.com/api/v1/events?discipline_uri=baseball&status=live"

# Season stages
curl "https://bf3zb3ipuy.us-east-1.awsapprunner.com/api/v1/seasons/mlb-2026/stages"

# Teams for the season
curl "https://bf3zb3ipuy.us-east-1.awsapprunner.com/api/v1/seasons/mlb-2026/entities"
```

---

## Competitors

All MLB events have two `team` competitors with `home` and `away` roles.

| Event Type | Competitor Type | Example |
|------------|----------------|---------|
| Match | `team` | San Diego Padres vs Los Angeles Dodgers |

---

## Entities

MLB team URIs are the normalised team name, e.g. `los-angeles-dodgers`. Each team entity has `discipline_uri` set to `baseball` to disambiguate from teams in other sports that share names (e.g. "Giants" in NFL vs MLB).

Teams carry their ESPN and SportsDataIO ids in `provider_ids`, and each game carries ESPN's game id in `espn_id`.

---

## Markets

MLB games can have the following market types:

| Type | Description | Example |
|------|-------------|---------|
| `moneyline` | Which team wins | Dodgers / Padres |
| `spread` | Run line (handicap) | Dodgers -1.5 / Padres +1.5 |
| `totals` | Over/under on total runs | Over 8.5 / Under 8.5 |

Odds are DraftKings lines as published by ESPN: `odds_source` is `espn-draftkings` and `odds_source_kind` is `book`.

---

## Data Freshness

- Games from 3 days ago to 14 days ahead are refreshed every 20 minutes.
- Yesterday's and today's games (US Eastern) are refreshed every 60 seconds, so a game's status and result change within about a minute.
- Once a game starts the posted line may stop moving; check each selection's `updated_at`.

---

## Event Status

MLB events use all standard event statuses plus `suspended`:

| Status | Description |
|--------|-------------|
| `upcoming` | Game not started |
| `live` | Game in progress |
| `completed` | Game finished |
| `cancelled` | Game will not be played |
| `postponed` | Game delayed (e.g. weather) |
| `suspended` | Game started but halted mid-game, to be resumed later |

The `suspended` status is specific to MLB, where games can be stopped and resumed on a different day.

---

## Example: MLB Game Event

```json
{
  "id": "a1b2c3d4-...",
  "name": "San Diego Padres vs Los Angeles Dodgers",
  "start_date": "2026-09-12T02:10:00Z",
  "status": "upcoming",
  "event_type": "match",
  "espn_id": "401817071",
  "category": { "uri": "sports", "name": "Sports" },
  "discipline": { "uri": "baseball", "name": "Baseball" },
  "competition": { "uri": "mlb", "name": "MLB", "country": "US", "discipline_uri": "baseball" },
  "season": { "uri": "mlb-2026", "name": "MLB 2026" },
  "stage": { "uri": "mlb-2026-regular-season", "name": "Regular Season", "stage_type": "regular-season", "stage_number": 2 },
  "competitors": [
    {
      "entity": { "uri": "los-angeles-dodgers", "type": "team", "name": "Los Angeles Dodgers", "discipline_uri": "baseball",
                  "provider_ids": { "espn": "19", "sportsdataio": "1" } },
      "role": "home"
    },
    {
      "entity": { "uri": "san-diego-padres", "type": "team", "name": "San Diego Padres", "discipline_uri": "baseball",
                  "provider_ids": { "espn": "25", "sportsdataio": "33" } },
      "role": "away"
    }
  ],
  "markets": [
    {
      "type": "moneyline",
      "selections": [
        { "outcome": "Los Angeles Dodgers", "odds_decimal": 1.67, "odds_source": "espn-draftkings", "odds_source_kind": "book" },
        { "outcome": "San Diego Padres", "odds_decimal": 2.30, "odds_source": "espn-draftkings", "odds_source_kind": "book" }
      ]
    },
    {
      "type": "spread",
      "selections": [
        { "outcome": "Los Angeles Dodgers", "odds_decimal": 1.91, "point": -1.5, "odds_source": "espn-draftkings", "odds_source_kind": "book" },
        { "outcome": "San Diego Padres", "odds_decimal": 1.91, "point": 1.5, "odds_source": "espn-draftkings", "odds_source_kind": "book" }
      ]
    },
    {
      "type": "totals",
      "selections": [
        { "outcome": "Over", "odds_decimal": 1.91, "point": 8.5, "odds_source": "espn-draftkings", "odds_source_kind": "book" },
        { "outcome": "Under", "odds_decimal": 1.91, "point": 8.5, "odds_source": "espn-draftkings", "odds_source_kind": "book" }
      ]
    }
  ]
}
```
