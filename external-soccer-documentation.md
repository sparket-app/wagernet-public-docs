# Soccer Data Model in WagerNet

How soccer data maps into WagerNet.

See [General Documentation](external-general-documentation.md) for general WagerNet API documentation.

---

## Data Hierarchy

```
Discipline: soccer
└── Competition: premier-league
    └── Season: premier-league-2026-27
        └── Stage: Regular Season (stage_type: regular-season)
            └── Event (event_type=match)
```

- **Competition** = one league or cup.
- **Season** = one edition of it. The uri carries one year where the season sits inside a
  calendar year (`mls-2026`) and two where it crosses one (`premier-league-2026-27`).
- **Stage** = the phase of the season. A domestic league has a single stage; a cup has one
  per round; MLS adds its playoff rounds.
- **Event** = one match. Soccer events are flat: no `parent_event_id`, no `child_events`.

### Competitions

| Competition | Name | `country` |
|---|---|---|
| `premier-league` | Premier League | England |
| `la-liga` | LALIGA | Spain |
| `bundesliga` | Bundesliga | Germany |
| `serie-a` | Serie A | Italy |
| `ligue-1` | Ligue 1 | France |
| `mls` | Major League Soccer | US |
| `champions-league` | UEFA Champions League | International |
| `europa-league` | UEFA Europa League | International |
| `fifa-world-cup` | FIFA World Cup | International |

`fifa-world-cup` is **archive data**, not a live feed: it came from a different provider,
its last price was written in July 2026, and nothing refreshes it. It is also the only
competition carrying a `to_advance` market. Everything else on this page describes the
eight live competitions.

### Stages differ by competition shape

| Shape | Stages | `stage_type` | Dates |
|---|---|---|---|
| Domestic leagues | one, `{season}-regular-season` | `regular-season` | absent |
| MLS | regular season plus its playoff rounds | `regular-season`, `wild-card`, `round-one`, `semifinals`, `conference-final`, `final` | absent |
| UEFA cups | one per round: `league-phase`, `knockout-round-playoffs`, `round-of-16`, `quarterfinals`, `semifinals`, `final` | **absent** | present |
| FIFA World Cup | none — its events carry no `stage` at all | — | — |

A cup's rounds are read from the provider as the draw is published, and that path does not
set `stage_type` — so **every cup stage has none**. Order cup rounds by `stage_number`
(1–6) or read `stage.uri`; don't branch on `stage_type` in soccer.

A cup season appears only once its draw is loaded, so a season row can exist with no events
and no roster yet.

---

## Quick Start

```bash
BASE=https://bf3zb3ipuy.us-east-1.awsapprunner.com

# Upcoming matches across every competition, 100 per page
curl "$BASE/api/v1/events?discipline_uri=soccer&status=upcoming&pageSize=100"

# One competition, one season
curl "$BASE/api/v1/events?competition_uri=premier-league&status=upcoming"
curl "$BASE/api/v1/events?season_uri=mls-2026&pageSize=100"

# One cup round
curl "$BASE/api/v1/events?stage_uri=champions-league-2026-27-league-phase&pageSize=100"

# The rounds of a cup season
curl "$BASE/api/v1/seasons/champions-league-2026-27/stages"

# The clubs seen in a season
curl "$BASE/api/v1/seasons/premier-league-2026-27/entities"
```

---

## Competitors

Every match has exactly two `team` competitors, one `role: "home"` and one `role: "away"`.

**The event `name` is `"<away> vs <home>"`** — away first. Never infer the home side from the
name; read `competitors[].role`.

---

## Entities

Clubs are entities of type `team` with `discipline_uri: "soccer"`. The uri is the club name
normalised, e.g. `seattle-sounders-fc`, `real-salt-lake`. Normalisation keeps characters
other than spaces, dots and apostrophes, so a uri can contain an ampersand
(`brighton-&-hove-albion`); World Cup entities are federation names in their own script.
Treat a uri as an opaque identifier and escape it if you ever put one in a URL — the API
addresses entities by `id`, not `uri`.

Each club carries one provider id, under the key `espn-soccer`:

```json
{ "uri": "seattle-sounders-fc", "type": "team", "discipline_uri": "soccer",
  "name": "Seattle Sounders FC", "provider_ids": { "espn-soccer": "9726" } }
```

That id space spans every soccer competition, so a club appearing in both a league and a cup
is the same entity. World Cup national teams carry no provider id.

`GET /api/v1/seasons/{uri}/entities` returns the **clubs** of a season, never players, and it
is built from whoever has appeared in an imported match — so it fills up as the season plays
and is empty for a season not yet imported.

---

## Markets

| Type | Selections | Where |
|---|---|---|
| `moneyline` | 3 — home, away and `Draw` | every match |
| `spread` | 2 — each team with a signed `point` | where priced |
| `totals` | 2 — `Over` / `Under` on total goals | where priced |
| `to_advance` | 2 — no draw | FIFA World Cup knockout matches only |

The moneyline is **three-way**: a draw is a real outcome, offered as a selection whose
`outcome` is `Draw`.

**A match can be listed before it is priced.** The moneyline is created with the match and
all three legs carry no `odds_decimal`, `odds_source` or `odds_source_kind` until a line
arrives — the keys are omitted, not null. Pricing is all-or-nothing per market: a moneyline
is never half priced. Spread and totals markets are absent entirely until priced, so an
event has between one and three markets.

Books post soccer lines only days before kick-off. Measured on 2026-09-10, every upcoming
match within a week of kick-off was priced, while matches eight days out and beyond were
about half unpriced — 28 of 158 upcoming matches had an unpriced moneyline.

### Odds sources

| `odds_source` | `odds_source_kind` | What it is |
|---|---|---|
| `espn-draftkings` | `book` | A DraftKings line, relayed by ESPN |
| `sparket.ai` | `model` | Model output fused with market prices, pushed in through the partner API |

World Cup selections predate odds provenance and carry no `odds_source` at all.

A partner price permanently outranks a book price on the same selection, so a market's
selections can read entirely `sparket.ai` while its odds history is mostly
`espn-draftkings` — the book's observations keep landing there. Filter odds history by
`source` to follow one of them:

```bash
curl "$BASE/api/v1/markets/{marketId}/odds-history?source=espn-draftkings"
```

Book prices are published with the margin removed; the posted price is `odds_decimal_raw` in
odds history only. Model quotes carry `confidence` and `run_id` instead.

---

## Event Status

| Status | Meaning |
|---|---|
| `upcoming` | Not kicked off |
| `live` | In progress |
| `completed` | Finished |
| `postponed` | Moved to a later date |
| `cancelled` | Will not be played; every selection is marked `void` |
| `suspended` | Abandoned or halted |

`suspended` can be stored but **cannot be passed to the `status` filter** — that request
returns 400. Fetch without the filter and select client-side.

---

## Results and Settlement

A completed match gains a `results` array, one entry per competitor, with the goals scored:

```json
"results": [
  { "entity_id": "bda991aa-...", "placement": 1, "score": { "goals": 2 } },
  { "entity_id": "1c60413c-...", "placement": 2, "score": { "goals": 0 } }
]
```

**A draw puts both clubs at `placement: 1`.** On the three-way moneyline the `Draw`
selection is then `won` and **both clubs read `lost`** — so a consumer that treats
"placement 1" as the winner will see two winners, and one that expects a winning team
selection on every settled match will find none.

| Market | How it settles |
|---|---|
| `moneyline` | One winner → that club `won`, the others `lost`. A draw → `Draw` `won`, both clubs `lost` |
| `spread` | Goals plus the selection's `point` against the opponent's goals; an exact tie is a `push` |
| `totals` | Both sides' goals summed against the `point`; exactly on the line is a `push` |
| `to_advance` | The side that progressed `won`, the other `lost` |

### Extra time and penalties

**Every market except `to_advance` settles on the score after ninety minutes.** Extra time
and a shootout do not count towards the moneyline, the spread or the total. A match won on
penalties is a **draw** for settlement: both sides at `placement: 1`, `Draw` `won`, both
clubs `lost`.

`to_advance` is the exception, and settles on who actually progressed — including extra time
and penalties. So on a knockout match decided by a shootout, `to_advance` names a winner
while the moneyline settles as a draw. That is not a contradiction: they answer two
different questions.

A match whose per-period score cannot be read is left **unresolved** rather than settled on
a total that includes extra time, so a settled market is late rather than wrong.

---

## Data Freshness

- Matches from 3 days ago to 14 days ahead are refreshed every 20 minutes.
- Yesterday's and today's matches (US Eastern) are refreshed every 60 seconds, so status,
  score and in-play odds change within about a minute.
- Cup seasons and their rounds are discovered twice a day.
- A season outside its own date window is not polled at all, and `fifa-world-cup` is not
  polled at all.

A price changes only when its source moves. Compare each selection's `updated_at` between
polls, and read the market's odds history for every move in between.

---

## Example: Upcoming League Match

```json
{
  "id": "c086363b-...",
  "name": "Real Salt Lake vs Seattle Sounders FC",
  "start_date": "2026-09-24T01:30:00Z",
  "status": "upcoming",
  "event_type": "match",
  "espn_id": "761543",
  "category": { "uri": "sports", "name": "Sports" },
  "discipline": { "uri": "soccer", "name": "Soccer", "category_uri": "sports" },
  "competition": { "uri": "mls", "name": "Major League Soccer", "country": "US",
                   "discipline_uri": "soccer" },
  "season": { "uri": "mls-2026", "name": "MLS 2026", "competition_uri": "mls" },
  "stage": { "uri": "mls-2026-regular-season", "name": "Regular Season",
             "stage_type": "regular-season", "stage_number": 1 },
  "competitors": [
    { "entity": { "uri": "seattle-sounders-fc", "type": "team", "name": "Seattle Sounders FC",
                  "discipline_uri": "soccer", "provider_ids": { "espn-soccer": "9726" } },
      "role": "home" },
    { "entity": { "uri": "real-salt-lake", "type": "team", "name": "Real Salt Lake",
                  "discipline_uri": "soccer", "provider_ids": { "espn-soccer": "4771" } },
      "role": "away" }
  ],
  "markets": [
    {
      "id": "27f89161-...",
      "type": "moneyline",
      "selections": [
        { "id": "8a94b372-...", "outcome": "Seattle Sounders FC", "is_current": true,
          "updated_at": "2026-09-09T00:24:20.270756Z" },
        { "id": "a9dc5cb8-...", "outcome": "Real Salt Lake", "is_current": true,
          "updated_at": "2026-09-09T00:24:20.278458Z" },
        { "id": "9099f531-...", "outcome": "Draw", "is_current": true,
          "updated_at": "2026-09-09T00:24:20.284240Z" }
      ]
    }
  ]
}
```

This match is more than a week out, so its moneyline exists with no price on any leg — each
selection carries only `id`, `outcome`, `is_current` and `updated_at`. It is also the event's
only market: the spread and totals markets do not exist yet.

## Example: Completed Match Drawn on the Day

```json
{
  "name": "AS Roma vs Fenerbahce",
  "status": "completed",
  "event_type": "match",
  "markets": [
    {
      "type": "moneyline",
      "selections": [
        { "outcome": "Fenerbahce", "odds_decimal": 3.6658, "result": "lost",
          "odds_source": "sparket.ai", "odds_source_kind": "model" },
        { "outcome": "AS Roma", "odds_decimal": 2.1421, "result": "lost",
          "odds_source": "sparket.ai", "odds_source_kind": "model" },
        { "outcome": "Draw", "odds_decimal": 3.8407, "result": "won",
          "odds_source": "sparket.ai", "odds_source_kind": "model" }
      ]
    }
  ],
  "results": [
    { "entity_id": "4e8ca3e6-...", "placement": 1, "score": { "goals": 1 } },
    { "entity_id": "93db0954-...", "placement": 1, "score": { "goals": 1 } }
  ]
}
```

Both clubs sit at `placement: 1`, both club selections read `lost`, and `Draw` is the only
winning selection.
