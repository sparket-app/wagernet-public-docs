# Tennis Data Model in WagerNet

How professional tennis data maps into WagerNet.

See [General Documentation](external-general-documentation.md) for general WagerNet API documentation.

---

## Data Hierarchy

```
Discipline: tennis
└── Competition: us-open-mens-singles        (one tournament draw)
    └── Season: us-open-mens-singles-2026    (one edition of that draw)
        ├── Stage: Qualifying 1st Round      (stage_type: qualifying, stage_number: 1)
        ├── Stage: Qualifying 2nd Round      (qualifying, 2)
        ├── Stage: Qualifying Final          (qualifying, 3)
        ├── Stage: Round 1                   (main, 4)
        ├── ...
        └── Stage: Final                     (main, 10)
            └── Event (event_type=match)
```

Tennis has no leagues. The unit of competition is a **draw** — one bracket inside one
tournament, with its own entry list and its own champion — so:

- **Competition** = one draw, not one tournament. Wimbledon is two competitions,
  `wimbledon-mens-singles` and `wimbledon-womens-singles`. The URI is the tournament name
  slugged plus the draw; the display `name` carries the draw too ("US Open Men's Singles").
- **Season** = one edition of that draw, `{competition_uri}-{year}`, e.g.
  `us-open-mens-singles-2026`. Its `start_date` / `end_date` are the tournament's own
  window, which covers qualifying week as well as the main draw.
- **Stage** = one round. Qualifying and the main bracket share one round vocabulary and one
  sequence, distinguished by `stage_type` (`qualifying` or `main`).
- **Event** = one match. Tennis events are flat: no `parent_event_id`, no `child_events`.

**Scope is singles on both tours** — men's and women's singles draws, qualifying included.
Doubles and mixed doubles draws are not imported, so there are no `pair` competitors in
tennis.

### Rounds

`stage_number` is WagerNet's own ordinal, chosen so that ordering by it runs qualifying
before the main draw:

| Stage name | `stage_number` | `stage_type` |
|---|---|---|
| Qualifying 1st Round | 1 | `qualifying` |
| Qualifying 2nd Round | 2 | `qualifying` |
| Qualifying Final | 3 | `qualifying` |
| Round 1 | 4 | `main` |
| Round 2 | 5 | `main` |
| Round 3 | 6 | `main` |
| Round 4 | 7 | `main` |
| Quarterfinal | 8 | `main` |
| Semifinal | 9 | `main` |
| Final | 10 | `main` |

Stages exist only for the rounds a given edition actually plays: a draw can carry
qualifying one year and not the next, and a 128-player bracket runs Round 3 straight into
the Quarterfinal with no Round 4. A stage's `start_date` and `end_date` are the first and
last match times in that round, so they tighten as the round is scheduled. A match whose
round is not published carries no `stage` at all.

### The competition list grows during the season

Competitions and seasons are read from the tour calendar rather than a fixed list. A
tournament publishes its draw about two weeks before it plays, and only then does its
competition and season appear — so the set of tennis competitions grows through the year,
and a consumer should discover it rather than hardcode it:

```bash
curl "https://bf3zb3ipuy.us-east-1.awsapprunner.com/api/v1/disciplines/tennis/competitions"
```

A competition's URI is minted once, when the draw is first seen, and never revisited. A
tournament renamed by a sponsor change keeps its URI and gets a new `name`; where two
tournaments would slug to the same URI, the second one's URI carries an extra numeric
segment. `country` is a country **name** read from the tournament venue — `United States`,
`Great Britain`, `China PR` — not an ISO code.

---

## Quick Start

```bash
BASE=https://bf3zb3ipuy.us-east-1.awsapprunner.com

# All tennis matches, newest schedule first, 100 per page
curl "$BASE/api/v1/events?discipline_uri=tennis&pageSize=100&sort=start_date:desc"

# Upcoming and in-progress matches
curl "$BASE/api/v1/events?discipline_uri=tennis&status=upcoming"
curl "$BASE/api/v1/events?discipline_uri=tennis&status=live"

# One draw, one edition, one round
curl "$BASE/api/v1/events?competition_uri=us-open-mens-singles"
curl "$BASE/api/v1/events?season_uri=us-open-mens-singles-2026&pageSize=100"
curl "$BASE/api/v1/events?stage_uri=us-open-mens-singles-2026-semifinal"

# The rounds of an edition
curl "$BASE/api/v1/seasons/us-open-mens-singles-2026/stages"

# Price history of one market, one source at a time
curl "$BASE/api/v1/markets/{marketId}/odds-history?source=tennis-model"
```

---

## Competitors

Every match has two `player` competitors with `home` and `away` roles. Home and away are
slot labels only: they carry no venue meaning in tennis, and there is no draw or tie.

| Event Type | Competitor Type | Example |
|---|---|---|
| Match | `player` | Frances Tiafoe vs Ben Shelton |

**A bracket slot with an unnamed side is not published as an event.** A two-way market
needs both names, so a match appears only once both players are known — including the final
while one semifinal is still to be played.

A small number of matches carry a **third competitor row**: a player who was in the
matchup and was later replaced. The event `name` and the `results` reflect the two who
played, the moneyline carries the extra outcome as a losing selection, and exactly one
selection is marked `won`. Five of the 2,151 tennis events held on 2026-09-10 had this
shape, so read the market and the results rather than assuming exactly two competitors.

---

## Entities

Tennis entities are all of type `player`, with `discipline_uri` set to `tennis`. The URI is
the normalised player name, e.g. `ben-shelton`, `karen-khachanov`; where two players share a
name, the second URI carries a numeric suffix. Each player carries the provider id under the
key `espn-tennis`:

```json
{ "uri": "ben-shelton", "type": "player", "discipline_uri": "tennis", "name": "Ben Shelton",
  "provider_ids": { "espn-tennis": "9250" } }
```

Each match also carries the provider's match id in `espn_id`.

Tennis has no season rosters — `GET /api/v1/seasons/{uri}/entities` returns an empty array.
Read the players off each event's `competitors`.

---

## Markets

Four market types, all two-way:

| Type | What it is | Selections |
|---|---|---|
| `moneyline` | Match winner | one per competitor, no `point` |
| `spread` | Sets handicap | favourite at `point: -1.5`, underdog at `+1.5` |
| `game_spread` | Games handicap over the whole match | favourite at `-5.5`, underdog at `+5.5` |
| `game_totals` | Total games played in the match | `Over` / `Under` at `21.5` (best of 3) or `35.5` (best of 5) |

There is no `totals` market on tennis: `spread` counts sets, and games are counted by the
two `game_*` markets.

**The lines never move.** The sets handicap is a set and a half on every match — winning
2-0 of three sets, or 3-0 / 3-1 of five — and the games handicap is five and a half games
whatever the format. Both are half-numbers, so neither handicap can push. The total is one
line per format.

**Only the moneyline is guaranteed.** It is created with the match; the other three appear
only where the match is priced. A match whose players cannot both be priced carries the
moneyline alone, with `odds_decimal: null` on its selections. Matches played over short sets
carry no games markets, since neither games line fits that format.

Which player is the favourite can change between ranking releases, which swaps the two
points on a handicap. The superseded pair is marked `is_current: false` rather than deleted,
so selection ids stay resolvable; pass `include_historical=true` to see them.

### Odds are model prices

Nobody posts a line on these matches: **there are no bookmaker and no exchange prices in
tennis at all.** Every price is computed, and both sources carry
`odds_source_kind: "model"`:

| `odds_source` | What it is |
|---|---|
| `tennis-model` | WagerNet's own price, derived from the weekly ATP / WTA tour ranking |
| `sparket.ai` | Model output fused with market prices, pushed in through the partner API |

Read `odds_source` **per selection**: one match can carry a `sparket.ai` moneyline next to a
`tennis-model` sets handicap. Where both sources have priced a selection, the current
`odds_decimal` is the `sparket.ai` one and the `tennis-model` series continues in the
market's odds history, so filter by `source` to compare them:

```bash
curl "$BASE/api/v1/markets/{marketId}/odds-history?source=tennis-model"
curl "$BASE/api/v1/markets/{marketId}/odds-history?source=sparket.ai"
```

`sparket.ai` quotes also carry `confidence` and `run_id` in odds history; `tennis-model`
quotes carry neither.

A player the tour ranking does not list cannot be priced, and a substitute rating is never
invented — so a match with an unranked player keeps an unpriced moneyline
(`odds_decimal: null`). This is common in qualifying, which is by definition the players
below the ranking line.

---

## Event Metadata

Match `metadata` carries up to three keys, all optional:

| Key | Type | Meaning |
|---|---|---|
| `best_of` | integer | Sets the match is played over: `3` or `5`. Absent where the format cannot be determined — it is never guessed. |
| `court` | string | Court the match is played on, e.g. `"Arthur Ashe Stadium"`, `"Court Kia"` |
| `ending` | string | `"retired"` or `"walkover"`. Present only on a match that ended that way. |

Five sets are played by the men's singles main draw of a Grand Slam, and by Wimbledon's
final qualifying round; every other match on tour is best of three. `best_of` is the field
to read for this — the number is derived from the draw, the round and the tournament, and
those inputs are not all exposed by the API.

`game_totals` follows `best_of` directly: `21.5` where it is 3, `35.5` where it is 5.

---

## Event Status

| Status | Meaning |
|---|---|
| `upcoming` | Not started |
| `live` | In progress |
| `completed` | Finished, including a retirement or a walkover |
| `cancelled` | Match will not be played; every selection on it is marked `void` |

A match halted for rain or darkness stays `live` and returns to play under the same status.
Tennis never reports `suspended` or `postponed`. On 2026-09-10 no tennis event was
`cancelled`.

### `start_date` is provisional

Tennis has no per-match clock time: only the first match on a court has one, and the rest
follow the match before them. `start_date` is stored as the tour publishes it and refreshed
as the schedule firms up, which means:

- every match of a round nobody has scheduled yet can share one timestamp — 12 matches sat
  on a single time in one Wimbledon qualifying round;
- the precise start is known only once the match is under way.

A consumer that needs a real cut-off should derive it from the round rather than treat
`start_date` as a deadline.

---

## Results

A completed match gains a `results` array with one entry per player, plus a tennis score in
each entry:

```json
"results": [
  { "entity_id": "18b097c5-...", "placement": 1,
    "score": { "sets": [ { "games": 6, "tiebreak": 5 }, { "games": 6, "tiebreak": 0 }, { "games": 6 },
                         { "games": 6 }, { "games": 7, "tiebreak": 10 } ],
               "sets_won": 3, "games_won": 31 } },
  { "entity_id": "0084f81b-...", "placement": 2,
    "score": { "sets": [ { "games": 7, "tiebreak": 7 }, { "games": 7, "tiebreak": 7 }, { "games": 0 },
                         { "games": 2 }, { "games": 6, "tiebreak": 5 } ],
               "sets_won": 2, "games_won": 22 } }
]
```

| Field | Meaning |
|---|---|
| `placement` | `1` for the winner, `2` for the loser. Tennis has no draw. |
| `score.sets[]` | Games won in each set, in order, with `tiebreak` giving the tie-break points where the set went to one. A tie-break set counts as seven games to whoever won it. |
| `score.sets_won` | Sets won — what the `spread` market settles against |
| `score.games_won` | Games won across the match — what `game_spread` and `game_totals` settle against |

A walkover has no sets played, so `sets` is `[]` and both scalars are `0` on each side.

### Retirements and walkovers

A retirement or a walkover is a `completed` match with a winner, and `metadata.ending` says
which it was.

- The **moneyline settles on the reported winner** — `won` / `lost`, not voided. Tennis
  always produces a winner.
- The **sets handicap, games handicap and total games are voided** — every selection on them
  gets `result: "void"`. The sets and games on the board when a match stops are not those of
  a contest, so a margin on them means nothing.

A match that ends without naming a winner is left alone: no `results`, and its selections
keep `result: null`.

---

## Data Freshness

- Every draw in progress is re-read and repriced about every 5 minutes, whole bracket each
  time — so a corrected start time, a new score or a settled market lands within a few
  minutes.
- The tour calendar is re-read about twice a day, which is when a newly published draw
  becomes a competition and a season.
- The tour rankings that `tennis-model` prices from are published weekly and checked daily.

Prices move only when their inputs move: a `tennis-model` price is rewritten when the
ranking behind it changes, so a market can sit unchanged for days. Compare each selection's
`updated_at` between polls, and read the market's odds history for every move in between.

---

## Example: Upcoming Match, All Four Markets

```json
{
  "id": "79eb23a6-...",
  "name": "Frances Tiafoe vs Ben Shelton",
  "start_date": "2026-09-11T23:00:00Z",
  "status": "upcoming",
  "event_type": "match",
  "espn_id": "182766",
  "category": { "uri": "sports", "name": "Sports" },
  "discipline": { "uri": "tennis", "name": "Tennis", "category_uri": "sports" },
  "competition": { "uri": "us-open-mens-singles", "name": "US Open Men's Singles",
                   "country": "United States", "discipline_uri": "tennis" },
  "season": { "uri": "us-open-mens-singles-2026", "name": "US Open Men's Singles 2026",
              "competition_uri": "us-open-mens-singles",
              "start_date": "2026-08-24T00:00:00Z", "end_date": "2026-09-13T00:00:00Z" },
  "stage": { "uri": "us-open-mens-singles-2026-semifinal", "name": "Semifinal",
             "season_uri": "us-open-mens-singles-2026", "stage_number": 9, "stage_type": "main" },
  "competitors": [
    { "entity": { "id": "9035286b-...", "uri": "frances-tiafoe", "type": "player",
                  "discipline_uri": "tennis", "name": "Frances Tiafoe",
                  "provider_ids": { "espn-tennis": "2708" } },
      "role": "home" },
    { "entity": { "id": "9e2d2612-...", "uri": "ben-shelton", "type": "player",
                  "discipline_uri": "tennis", "name": "Ben Shelton",
                  "provider_ids": { "espn-tennis": "9250" } },
      "role": "away" }
  ],
  "markets": [
    {
      "id": "5d95d790-...",
      "type": "moneyline",
      "selections": [
        { "id": "64f8aed0-...", "outcome": "Frances Tiafoe", "odds_decimal": 3.5214,
          "is_current": true, "odds_source": "sparket.ai", "odds_source_kind": "model" },
        { "id": "a1c9fc1f-...", "outcome": "Ben Shelton", "odds_decimal": 1.3966,
          "is_current": true, "odds_source": "sparket.ai", "odds_source_kind": "model" }
      ]
    },
    {
      "id": "a320b957-...",
      "type": "spread",
      "selections": [
        { "id": "c00c1e13-...", "outcome": "Ben Shelton", "odds_decimal": 2.61, "point": -1.5,
          "is_current": true, "odds_source": "tennis-model", "odds_source_kind": "model" },
        { "id": "217351b5-...", "outcome": "Frances Tiafoe", "odds_decimal": 1.62, "point": 1.5,
          "is_current": true, "odds_source": "tennis-model", "odds_source_kind": "model" }
      ]
    },
    {
      "id": "0ecfe70b-...",
      "type": "game_spread",
      "selections": [
        { "id": "0e743520-...", "outcome": "Ben Shelton", "odds_decimal": 3.2, "point": -5.5,
          "is_current": true, "odds_source": "tennis-model", "odds_source_kind": "model" },
        { "id": "8fd932c5-...", "outcome": "Frances Tiafoe", "odds_decimal": 1.45, "point": 5.5,
          "is_current": true, "odds_source": "tennis-model", "odds_source_kind": "model" }
      ]
    },
    {
      "id": "24f35141-...",
      "type": "game_totals",
      "selections": [
        { "id": "fc4e317f-...", "outcome": "Over", "odds_decimal": 1.5, "point": 35.5,
          "is_current": true, "odds_source": "tennis-model", "odds_source_kind": "model" },
        { "id": "26358c06-...", "outcome": "Under", "odds_decimal": 3.02, "point": 35.5,
          "is_current": true, "odds_source": "tennis-model", "odds_source_kind": "model" }
      ]
    }
  ],
  "metadata": { "best_of": 5, "court": "Arthur Ashe Stadium" }
}
```

## Example: Completed Match Ended by Retirement

The moneyline is settled on the winner; the sets handicap is voided. Trimmed to the parts
that differ from the example above.

```json
{
  "id": "6070b6a9-...",
  "name": "Karen Khachanov vs Alexander Blockx",
  "start_date": "2026-09-09T21:05:00Z",
  "status": "completed",
  "event_type": "match",
  "stage": { "uri": "us-open-mens-singles-2026-quarterfinal", "name": "Quarterfinal",
             "stage_number": 8, "stage_type": "main" },
  "markets": [
    {
      "type": "moneyline",
      "selections": [
        { "outcome": "Karen Khachanov", "odds_decimal": 1.6086, "result": "won",
          "odds_source": "sparket.ai", "odds_source_kind": "model" },
        { "outcome": "Alexander Blockx", "odds_decimal": 2.6432, "result": "lost",
          "odds_source": "sparket.ai", "odds_source_kind": "model" }
      ]
    },
    {
      "type": "spread",
      "selections": [
        { "outcome": "Alexander Blockx", "odds_decimal": 2.57, "point": -1.5, "result": "void",
          "odds_source": "tennis-model", "odds_source_kind": "model" },
        { "outcome": "Karen Khachanov", "odds_decimal": 1.64, "point": 1.5, "result": "void",
          "odds_source": "tennis-model", "odds_source_kind": "model" }
      ]
    }
  ],
  "results": [
    { "entity_id": "d8ae4cff-...", "placement": 1,
      "score": { "sets": [ { "games": 6 }, { "games": 7 }, { "games": 3 } ], "sets_won": 3, "games_won": 16 } },
    { "entity_id": "693e1423-...", "placement": 2,
      "score": { "sets": [ { "games": 2 }, { "games": 5 }, { "games": 2 } ], "sets_won": 0, "games_won": 9 } }
  ],
  "metadata": { "best_of": 5, "court": "Arthur Ashe Stadium", "ending": "retired" }
}
```
