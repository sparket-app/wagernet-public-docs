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
            └── Event (event_type=game)
```

- **Stage** = season phase. Stages are ordered by `stage_number` (1 = Spring Training through 6 = World Series).
- **Event** = individual game between two teams.

MLB events are flat (no parent-child nesting). Each game is a standalone event.

---

## Quick Start

```bash
# All baseball events
curl "https://bf3zb3ipuy.us-east-1.awsapprunner.com/api/v1/events?discipline_uri=baseball"

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
| Game | `team` | New York Yankees vs Boston Red Sox |

---

## Entities

MLB team entities use a `sportsdataio-mlb-` prefix for global uniqueness. Each team entity has `discipline_uri` set to `baseball` to disambiguate from teams in other sports that share names (e.g. "Giants" in NFL vs MLB).

Teams are created automatically during schedule import.

---

## Markets

MLB games can have the following market types:

| Type | Description | Example |
|------|-------------|---------|
| `moneyline` | Which team wins | Yankees -150 / Red Sox +130 |
| `spread` | Run line (handicap) | Yankees -1.5 / Red Sox +1.5 |
| `totals` | Over/under on total runs | Over 8.5 / Under 8.5 |

Odds are sourced from SportsDataIO and reflect the configured sportsbook (e.g. DraftKings).

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

## Admin Endpoints

MLB data is imported through two admin endpoints:

### Sync Season (Schedule + Teams)

```
GET /api/v1/admin/mlb/sync/season/{seasonURI}
```

Imports the full MLB schedule for a season, including all teams and games. Teams are created as entities if they don't already exist.

**Path parameters:**

| Parameter | Description | Example |
|-----------|-------------|---------|
| `seasonURI` | Season URI | `mlb-2026` |

**Query parameters:**

| Parameter | Description | Default |
|-----------|-------------|---------|
| `includeSpringTraining` | Include spring training games | `false` |

Example:
```bash
curl "https://bf3zb3ipuy.us-east-1.awsapprunner.com/api/v1/admin/mlb/sync/season/mlb-2026"
curl "https://bf3zb3ipuy.us-east-1.awsapprunner.com/api/v1/admin/mlb/sync/season/mlb-2026?includeSpringTraining=true"
```

### Sync Odds

```
GET /api/v1/admin/mlb/sync/odds?dateStart=...&dateEnd=...
```

Imports MLB odds (moneyline, spread, totals) for games within a date range.

**Query parameters:**

| Parameter | Description | Example |
|-----------|-------------|---------|
| `dateStart` | Start date (YYYY-MM-DD) | `2026-04-01` |
| `dateEnd` | End date (YYYY-MM-DD) | `2026-04-07` |

Example:
```bash
curl "https://bf3zb3ipuy.us-east-1.awsapprunner.com/api/v1/admin/mlb/sync/odds?dateStart=2026-04-01&dateEnd=2026-04-07"
```

Both endpoints return the standard import result object with counts of events created, updated, markets synced, etc.

---

## Background Sync Jobs

When enabled, WagerNet runs three background jobs to keep MLB data current:

| Job | Interval | Description |
|-----|----------|-------------|
| MLB Schedule Sync | Daily | Re-imports the full season schedule to pick up makeup games and schedule changes |
| MLB Results Sync | 60 seconds | Polls today and yesterday (Eastern Time) for live score updates and game results |
| MLB Full Sync | 20 minutes | Syncs odds and results for games from 3 days ago to 14 days ahead |

These jobs are opt-in via environment variables (`MLB_SCHEDULE_SYNC_ENABLED`, `MLB_RESULTS_SYNC_ENABLED`, `MLB_SYNC_ENABLED`).

---

## Example: MLB Game Event

```json
{
  "id": "a1b2c3d4-...",
  "name": "New York Yankees vs Boston Red Sox",
  "start_date": "2026-04-05T23:10:00Z",
  "status": "upcoming",
  "event_type": "game",
  "category": { "uri": "sports", "name": "Sports" },
  "discipline": { "uri": "baseball", "name": "Baseball" },
  "competition": { "uri": "mlb", "name": "MLB", "country": "US", "discipline_uri": "baseball" },
  "season": { "uri": "mlb-2026", "name": "MLB 2026" },
  "stage": { "uri": "mlb-2026-regular-season", "name": "Regular Season", "stage_type": "regular-season", "stage_number": 2 },
  "competitors": [
    {
      "entity": { "uri": "sportsdataio-mlb-nyy", "type": "team", "name": "New York Yankees", "discipline_uri": "baseball" },
      "role": "home"
    },
    {
      "entity": { "uri": "sportsdataio-mlb-bos", "type": "team", "name": "Boston Red Sox", "discipline_uri": "baseball" },
      "role": "away"
    }
  ],
  "markets": [
    {
      "type": "moneyline",
      "selections": [
        { "outcome": "New York Yankees", "odds_decimal": 1.67 },
        { "outcome": "Boston Red Sox", "odds_decimal": 2.30 }
      ]
    },
    {
      "type": "spread",
      "selections": [
        { "outcome": "New York Yankees", "odds_decimal": 1.91, "point": -1.5 },
        { "outcome": "Boston Red Sox", "odds_decimal": 1.91, "point": 1.5 }
      ]
    },
    {
      "type": "totals",
      "selections": [
        { "outcome": "Over", "odds_decimal": 1.91, "point": 8.5 },
        { "outcome": "Under", "odds_decimal": 1.91, "point": 8.5 }
      ]
    }
  ],
  "results": []
}
```
