# American Football Data Model in WagerNet

How NFL and college football data maps into WagerNet.

See [General Documentation](external-general-documentation.md) for general WagerNet API documentation.

---

## Data Hierarchy

```
Discipline: american-football
├── Competition: nfl        (NFL)
│   └── Season: nfl-2026
│       ├── Stage: Preseason Week 1   (stage_type: preseason, stage_number: -3)
│       ├── Stage: Week 1             (regular, 1)
│       └── Stage: Super Bowl         (superbowl, 22)
│           └── Event (event_type=match)
└── Competition: ncaaf     (College Football)
    └── Season: ncaaf-2026
        ├── Stage: Week 1             (regular, 1)
        └── Stage: Bowls & Playoff    (postseason, 100)
            └── Event (event_type=match)
```

- **Competition** = `nfl` or `ncaaf`, both with `country: "US"`.
- **Season** = one year, `nfl-2026` / `ncaaf-2026`.
- **Stage** = one week or one postseason round.
- **Event** = one game. Events are flat: no `parent_event_id`, no `child_events`.

### Stages and their ordering

`stage_number` is a sort key built so that ascending order is chronological. It is **not** a
week number in every case: NFL preseason numbers are **negative**, and the whole college
postseason is 100. Sort by it; don't parse the uri.

| Competition | Stage uri | `stage_type` | `stage_number` |
|---|---|---|---|
| `nfl` | `nfl-{year}-hall-of-fame` | `preseason` | -4 |
| `nfl` | `nfl-{year}-preseason-{1..3}` | `preseason` | -3 … -1 |
| `nfl` | `nfl-{year}-week-{1..18}` | `regular` | 1 … 18 |
| `nfl` | `nfl-{year}-wildcard` | `wildcard` | 19 |
| `nfl` | `nfl-{year}-divisional` | `divisional` | 20 |
| `nfl` | `nfl-{year}-conference` | `conference` | 21 |
| `nfl` | `nfl-{year}-superbowl` | `superbowl` | 22 |
| `ncaaf` | `ncaaf-{year}-week-{n}` | `regular` | n |
| `ncaaf` | `ncaaf-{year}-bowls` | `postseason` | 100 |

Note the regular-season `stage_type` here is `regular`, not the `regular-season` used by
other sports.

College football has **no preseason**, and its bowls and playoff games all share the single
`bowls` stage.

**A stage exists only once its games have been imported.** Weeks beyond the current sync
window, and the playoff or bowl stages, simply do not exist as rows yet — a season's stage
list grows through the year. The NFL Pro Bowl is never imported.

---

## Quick Start

```bash
BASE=https://bf3zb3ipuy.us-east-1.awsapprunner.com

# This season's upcoming games, either competition
curl "$BASE/api/v1/events?competition_uri=nfl&status=upcoming&pageSize=100"
curl "$BASE/api/v1/events?competition_uri=ncaaf&status=upcoming&pageSize=100"

# One week
curl "$BASE/api/v1/events?stage_uri=nfl-2026-week-3&pageSize=100"

# The weeks that exist so far
curl "$BASE/api/v1/seasons/nfl-2026/stages"

# The teams seen this season
curl "$BASE/api/v1/seasons/nfl-2026/entities"

# How one market's line moved, book prices only
curl "$BASE/api/v1/markets/{marketId}/odds-history?source=espn-draftkings"
```

---

## Competitors

Every game has exactly two `team` competitors, one `role: "home"` and one `role: "away"`.

**The event `name` is `"<away> vs <home>"`** — away first. Read `competitors[].role` rather
than the name to tell who is at home.

---

## Entities

Teams are entities of type `team` with `discipline_uri: "american-football"`. The uri is the
team's display name normalised — `chicago-bears`, `tcu-horned-frogs`. Normalisation keeps
characters other than spaces, dots and apostrophes, so some college uris contain an
ampersand or brackets (`texas-a&m-aggies`, `miami-(oh)-redhawks`). Treat a uri as an opaque
identifier and escape it if you put one in a URL; the API addresses entities by `id`.

Provider ids use **two separate namespaces**, because the provider's NFL and college id
spaces overlap:

| Key | Used for |
|---|---|
| `espn-nfl` | NFL teams |
| `espn-ncaaf` | College teams |

```json
{ "uri": "chicago-bears", "type": "team", "discipline_uri": "american-football",
  "name": "Chicago Bears", "provider_ids": { "espn-nfl": "3" } }
```

`GET /api/v1/seasons/{uri}/entities` returns the season's teams. It is built from whoever has
appeared in an imported game, not from a league roster — so the college list is larger than
the top-division field, because lower-division opponents get added when they play one.

---

## Markets

| Type | Selections | `point` |
|---|---|---|
| `moneyline` | 2, one per team | none |
| `spread` | 2, one per team | signed and mirrored: home `-3`, away `+3` |
| `totals` | 2, `Over` / `Under` | the same value on both |

**There is no draw selection.** American football moneylines are two-way even though a tie
is possible — see Settlement below for what that means when a game ties.

**A moneyline can exist before it is priced.** It is created with the game, and its two
selections then carry no `odds_decimal`, `odds_source` or `odds_source_kind` at all — the
keys are omitted, not null. This is a college phenomenon: books post those lines late, and
on 2026-09-10 about a fifth of college moneyline selections were unpriced while no NFL
selection was. Spread and totals markets are absent entirely until priced, so an event
carries between one and three markets.

Lines are wide in college — spreads run past 50 points — and narrower in the NFL.

### Odds sources

| `odds_source` | `odds_source_kind` | What it is |
|---|---|---|
| `espn-draftkings` | `book` | A DraftKings line, relayed by ESPN |
| `sparket.ai` | `model` | Model output fused with market prices, pushed in through the partner API |

Most current prices in this discipline read `sparket.ai`, because a partner price
permanently outranks a book price on the same selection. The book's observations keep
landing in odds history, so filter history by `source` to follow the posted line.

Book prices are published with the margin removed — the implied probabilities of a market
sum to 1 — so `odds_decimal` will not match the book's screen price. The posted price is
`odds_decimal_raw`, in odds history only. Model quotes carry `confidence` and `run_id`
instead.

A market can hold a **ladder of lines**: the current one is what you get by default, while
superseded lines stay as `is_current: false` selections, each with its own `point` and its
own settled `result`. Pass `include_historical=true` to see them, and filter odds history by
`point` to follow one line. A market whose rows are all superseded comes back with an empty
`selections` array.

---

## Event Metadata

American football events carry **no `metadata`** — the field is absent. Only `espn_id` is
published as a cross-provider key.

---

## Event Status

| Status | Meaning |
|---|---|
| `upcoming` | Not kicked off |
| `live` | In progress |
| `completed` | The provider has moved the game to its final state |
| `postponed` | Moved to a later date |
| `cancelled` | Will not be played; every selection is marked `void` |

`completed` follows the provider's own game state rather than a clock or the presence of
results.

---

## Results and Settlement

A completed game gains a `results` array, one entry per team, carrying the final points:

```json
"results": [
  { "entity_id": "eb610dfd-...", "placement": 1, "score": { "points": 13 } },
  { "entity_id": "0084f81b-...", "placement": 2, "score": { "points": 10 } }
]
```

There is no quarter-by-quarter breakdown — `points` is the whole score.

| Market | How it settles |
|---|---|
| `moneyline` | The winner `won`, the loser `lost` |
| `spread` | The team's points plus the selection's `point` against the opponent's points; landing exactly on the line is a `push` |
| `totals` | Both teams' points summed against the `point`; exactly on the line is a `push` |

**A tie puts both teams at `placement: 1`, and the moneyline pushes on both sides** — there
is no draw selection to win, so nobody wins. NFL regular-season games can legally end tied.

Whole-number spreads and totals push regularly. A real example: Seattle beat New England
13-10 with the spread at 3, so both spread selections read `push` while the total at 46.5
settled `Under`.

---

## Data Freshness

- Games in the current week are refreshed every 60 seconds, so status, score and in-play
  odds change within about a minute.
- The window from last week to two weeks ahead is refreshed every 20 minutes.
- **Anything further out than two weeks ahead is not refreshed at all** until the window
  reaches it — and the stages for those weeks do not exist yet.
- Out of season nothing updates.

A price changes only when its source moves. Compare each selection's `updated_at` between
polls, and read the market's odds history for every move in between.

---

## Example: Upcoming NFL Game

```json
{
  "id": "064368f9-...",
  "name": "Philadelphia Eagles vs Chicago Bears",
  "start_date": "2026-09-29T00:15:00Z",
  "status": "upcoming",
  "event_type": "match",
  "espn_id": "401872963",
  "category": { "uri": "sports", "name": "Sports" },
  "discipline": { "uri": "american-football", "name": "American Football",
                  "category_uri": "sports" },
  "competition": { "uri": "nfl", "name": "NFL", "country": "US",
                   "discipline_uri": "american-football" },
  "season": { "uri": "nfl-2026", "name": "NFL 2026", "competition_uri": "nfl" },
  "stage": { "uri": "nfl-2026-week-3", "name": "Week 3", "stage_type": "regular",
             "stage_number": 3 },
  "competitors": [
    { "entity": { "uri": "chicago-bears", "type": "team", "name": "Chicago Bears",
                  "discipline_uri": "american-football",
                  "provider_ids": { "espn-nfl": "3" } },
      "role": "home" },
    { "entity": { "uri": "philadelphia-eagles", "type": "team", "name": "Philadelphia Eagles",
                  "discipline_uri": "american-football",
                  "provider_ids": { "espn-nfl": "21" } },
      "role": "away" }
  ],
  "markets": [
    {
      "id": "f761a259-...",
      "type": "moneyline",
      "selections": [
        { "outcome": "Chicago Bears", "odds_decimal": 1.9123,
          "odds_source": "sparket.ai", "odds_source_kind": "model" },
        { "outcome": "Philadelphia Eagles", "odds_decimal": 2.0961,
          "odds_source": "sparket.ai", "odds_source_kind": "model" }
      ]
    },
    {
      "id": "a987f569-...",
      "type": "spread",
      "selections": [
        { "outcome": "Chicago Bears", "odds_decimal": 1.9919, "point": -1.5,
          "odds_source": "sparket.ai", "odds_source_kind": "model" },
        { "outcome": "Philadelphia Eagles", "odds_decimal": 2.0081, "point": 1.5,
          "odds_source": "sparket.ai", "odds_source_kind": "model" }
      ]
    },
    {
      "id": "c949f6d1-...",
      "type": "totals",
      "selections": [
        { "outcome": "Over", "odds_decimal": 2.1268, "point": 46.5,
          "odds_source": "sparket.ai", "odds_source_kind": "model" },
        { "outcome": "Under", "odds_decimal": 1.8875, "point": 46.5,
          "odds_source": "sparket.ai", "odds_source_kind": "model" }
      ]
    }
  ]
}
```

## Example: Completed Game, Spread Pushed

```json
{
  "name": "New England Patriots vs Seattle Seahawks",
  "status": "completed",
  "event_type": "match",
  "markets": [
    {
      "type": "moneyline",
      "selections": [
        { "outcome": "Seattle Seahawks", "result": "won" },
        { "outcome": "New England Patriots", "result": "lost" }
      ]
    },
    {
      "type": "spread",
      "selections": [
        { "outcome": "Seattle Seahawks", "point": -3, "result": "push" },
        { "outcome": "New England Patriots", "point": 3, "result": "push" }
      ]
    },
    {
      "type": "totals",
      "selections": [
        { "outcome": "Over", "point": 44.5, "result": "lost" },
        { "outcome": "Under", "point": 44.5, "result": "won" }
      ]
    }
  ],
  "results": [
    { "entity_id": "eb610dfd-...", "placement": 1, "score": { "points": 13 } },
    { "entity_id": "0084f81b-...", "placement": 2, "score": { "points": 10 } }
  ]
}
```
