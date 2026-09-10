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

Teams carry both providers' ids in `provider_ids`, under the keys `espn` and `sportsdataio`:

```json
{ "uri": "chicago-white-sox", "type": "team", "discipline_uri": "baseball",
  "name": "Chicago White Sox", "provider_ids": { "espn": "4", "sportsdataio": "16" } }
```

Those ids and `discipline_uri` appear only on the entity **embedded in an event's
`competitors`**. The season-roster and single-entity endpoints return a reduced view —
`id`, `uri`, `type` and `name` only — so read a team's provider ids off an event.

Games carry a provider's own game id, but **which one depends on where the game came from**:
games imported from ESPN carry `espn_id`, and the earlier SportsDataIO-era games carry
`sportsdataio_id` instead. No game carries both, so check for the key you need rather than
assuming `espn_id`.

---

## Markets

MLB games can have the following market types:

| Type | Description | Example |
|------|-------------|---------|
| `moneyline` | Which team wins | White Sox / Pirates |
| `spread` | Run line (handicap) | Pirates -1.5 / White Sox +1.5 |
| `totals` | Over/under on total runs | Over 7.5 / Under 7.5 |

Current odds come from DraftKings by way of ESPN: `odds_source` is `espn-draftkings` and
`odds_source_kind` is `book`.

**The published price has the book's margin removed.** A market's implied probabilities sum
to exactly 1, so `odds_decimal` will not match DraftKings' screen price — a two-way market
sits near 1.99 / 2.01 rather than 1.91 / 1.91. The price as posted survives only in odds
history, as `odds_decimal_raw`.

Two gaps worth handling:

- **Older games carry priced selections with no stated source.** Games from the
  SportsDataIO era keep their odds but have no `odds_source` and no `odds_source_kind`, and
  those prices still carry the book's margin. A price without a stated origin is not
  evidence of one.
- **Spring training games carry no markets at all**, even though they are events like any
  other.

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

`suspended` matters most in baseball, where a game can be stopped and resumed on another
day, but it is not exclusive to it. Note that it cannot be passed to the `status` filter —
that request returns 400 — so fetch without the filter and select client-side.

---

## Event Metadata

Games imported from ESPN carry no `metadata`. The earlier SportsDataIO-era games carry:

| Key | Meaning |
|---|---|
| `provider_status` | The source's own status string, e.g. `Final`, `Scheduled`, `Postponed` |
| `starting_pitchers` | `{ "home": name, "away": name }` |
| `rescheduled_game_id` / `rescheduled_from_game_id` | Links a postponed game to its replacement |
| `suspension_resume_date` | When a suspended game resumes |

---

## Results and Settlement

A completed game gains a `results` array, one entry per team:

```json
"results": [
  { "entity_id": "d38a42dd-...", "placement": 1, "score": { "runs": 2 } },
  { "entity_id": "eceba879-...", "placement": 2, "score": { "runs": 0 } }
]
```

`placement` is 1 for the winner and 2 for the loser. The score's `runs` is what every market
settles against; older games also carry `hits` and `errors`.

| Market | How it settles |
|---|---|
| `moneyline` | The winner `won`, the loser `lost` |
| `spread` | The team's runs plus the selection's `point` against the opponent's runs; landing exactly on the line is a `push` |
| `totals` | Both teams' runs summed against the `point`; exactly on the line is a `push` |

Run lines and totals are usually half-numbers, which cannot push.

---

## Example: MLB Game Event

```json
{
  "id": "7ea686a7-...",
  "name": "Pittsburgh Pirates vs Chicago White Sox",
  "start_date": "2026-09-10T23:40:00Z",
  "status": "upcoming",
  "event_type": "match",
  "espn_id": "401816886",
  "category": { "uri": "sports", "name": "Sports" },
  "discipline": { "uri": "baseball", "name": "Baseball" },
  "competition": { "uri": "mlb", "name": "MLB", "country": "US", "discipline_uri": "baseball" },
  "season": { "uri": "mlb-2026", "name": "MLB 2026" },
  "stage": { "uri": "mlb-2026-regular-season", "name": "Regular Season", "stage_type": "regular-season", "stage_number": 2 },
  "competitors": [
    {
      "entity": { "uri": "chicago-white-sox", "type": "team", "name": "Chicago White Sox", "discipline_uri": "baseball",
                  "provider_ids": { "espn": "4", "sportsdataio": "16" } },
      "role": "home"
    },
    {
      "entity": { "uri": "pittsburgh-pirates", "type": "team", "name": "Pittsburgh Pirates", "discipline_uri": "baseball",
                  "provider_ids": { "espn": "23", "sportsdataio": "4" } },
      "role": "away"
    }
  ],
  "markets": [
    {
      "id": "4ed403c7-...",
      "type": "moneyline",
      "selections": [
        { "id": "1434e1c0-...", "outcome": "Chicago White Sox", "odds_decimal": 1.9948, "is_current": true,
          "odds_source": "espn-draftkings", "odds_source_kind": "book", "updated_at": "2026-09-10T17:56:53.122995Z" },
        { "id": "cb508e47-...", "outcome": "Pittsburgh Pirates", "odds_decimal": 2.0053, "is_current": true,
          "odds_source": "espn-draftkings", "odds_source_kind": "book", "updated_at": "2026-09-10T17:56:53.136541Z" }
      ]
    },
    {
      "id": "f6d5d80b-...",
      "type": "spread",
      "selections": [
        { "outcome": "Chicago White Sox", "odds_decimal": 1.6024, "point": 1.5, "is_current": true,
          "odds_source": "espn-draftkings", "odds_source_kind": "book" },
        { "outcome": "Pittsburgh Pirates", "odds_decimal": 2.6601, "point": -1.5, "is_current": true,
          "odds_source": "espn-draftkings", "odds_source_kind": "book" }
      ]
    },
    {
      "id": "f8e2803d-...",
      "type": "totals",
      "selections": [
        { "outcome": "Over", "odds_decimal": 1.9293, "point": 7.5, "is_current": true,
          "odds_source": "espn-draftkings", "odds_source_kind": "book" },
        { "outcome": "Under", "odds_decimal": 2.0761, "point": 7.5, "is_current": true,
          "odds_source": "espn-draftkings", "odds_source_kind": "book" }
      ]
    }
  ]
}
```

The moneyline reads 1.9948 / 2.0053 rather than a book's 1.91 / 1.91 because the margin has
been removed: the two implied probabilities sum to exactly 1.
