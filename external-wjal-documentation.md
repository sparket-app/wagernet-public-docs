# WJAL Data Model in WagerNet

How World Jai-Alai League (Battle Court at JAM Arena, Miami) data maps into WagerNet.

See [General Documentation](external-general-documentation.md) for general WagerNet API documentation.

---

## Data Hierarchy

```
Discipline: jai-alai
└── Competition: wjal
    └── Season: wjal-fall-2026
        ├── Stage: Regular Season   (uri: wjal-fall-2026-regular,      stage_type: regular)
        ├── Stage: Playoffs         (uri: wjal-fall-2026-playoffs,     stage_type: playoffs)
        └── Stage: Championship     (uri: wjal-fall-2026-championship, stage_type: championship)
            └── Performance (parent event, event_type=performance)
                ├── Match 1 (Doubles) (child event, event_type=match)
                ├── Match 2 (Doubles)
                ├── ...
                └── Match 6 (Doubles)
```

- **Stage** = season phase (Regular Season, Playoffs, Championship). Note the regular-season
  `stage_type` is `regular`, not the `regular-season` used by other sports.
- **Performance** = game day (two teams, 6-7 matches)
- **Match** = individual match (child event linked via `parent_event_id`)

There are two seasons a year, Spring and Fall, named `wjal-spring-{year}` and
`wjal-fall-{year}`. A season's rows exist from the moment it is created, so the current
season can be listed with no events and an incomplete roster until its schedule and
rankings are published. Read the season list rather than hardcoding a season:

```bash
curl "https://bf3zb3ipuy.us-east-1.awsapprunner.com/api/v1/competitions/wjal/seasons"
```

---

## Quick Start

```bash
BASE=https://bf3zb3ipuy.us-east-1.awsapprunner.com

# The seasons, newest first — pick one before querying below
curl "$BASE/api/v1/competitions/wjal/seasons"

# All jai-alai events, 100 per page
curl "$BASE/api/v1/events?discipline_uri=jai-alai&pageSize=100"

# Just the game days, or just the matches
curl "$BASE/api/v1/events?discipline_uri=jai-alai&type=performance"
curl "$BASE/api/v1/events?discipline_uri=jai-alai&type=match"

# Season stages
curl "$BASE/api/v1/seasons/wjal-spring-2026/stages"

# Team rosters (players + pairs)
curl "$BASE/api/v1/seasons/wjal-spring-2026/entities"
```

---

## Competitors

| Event Type | Competitor Type | Example |
|------------|----------------|---------|
| Performance | `team` | Chargers vs Devils |
| Match (doubles) | `pair` with nested `player` members | Iturbide & Ubilla vs Benny & Etcheberry |
| Match (singles) | `player` | Bradley vs Jeden |

To find which team a match competitor belongs to, look up the parent performance via `parent_event_id` and match the `role` (home/away).

---

## Teams

| Team `name` | URI |
|------|-----|
| Devils | `wjal-devils` |
| Warriors | `wjal-warriors` |
| Chargers | `wjal-chargers` |
| Cyclones | `wjal-cyclones` |
| Fireballs | `wjal-fireballs` |
| Renegades | `wjal-renegades` |

---

## Entity URIs

All WJAL entities use a `wjal-` prefix for global uniqueness:

| Type | Pattern | Example |
|------|---------|---------|
| team | `wjal-{name}` | `wjal-devils` |
| player | `wjal-{stagename}-{id}` | `wjal-iturbide-44` |
| pair | `wjal-pair-{p1}-{p2}` | `wjal-pair-iturbide-44-ubilla-49` |

---

## Match Metadata

| Field | Description | Example |
|-------|-------------|---------|
| `match_number` | Position in game day (1-7) | `1` |
| `match_type` | Singles (`S`) or Doubles (`D`) | `"D"` |

---

## Divisions & Rankings

Each season, WJAL assigns every player and pair a **division** (matchup slot) and **rank** (strength rating):

- **Division** — determines who plays whom. Same-division players from different teams face each other. Singles have divisions 1-5, doubles have divisions 1-6.
- **Ranking** — skill rating within a division across all 6 teams. 1 = strongest, 6 = weakest.

Division and ranking are published per season, and a season carries none until its rankings
are set — a newly created season's roster lists its teams and members with no `division` or
`ranking` at all. Even within a ranked season some members carry neither: 17 of the 83
members of `wjal-spring-2026` have no division or ranking. A member without a ranking is
priced as a mid-table one rather than skipped.

Division and ranking are available on the entities endpoint:

```bash
curl "https://bf3zb3ipuy.us-east-1.awsapprunner.com/api/v1/seasons/wjal-spring-2026/entities"
```

Each team member has `division` and `ranking` fields:

```json
{
  "entity": { "uri": "wjal-goixerri-40", "type": "player", "name": "Goixerri" },
  "division": 1,
  "ranking": 1
}
```

---

## Moneyline Markets & Odds

Both match and performance events have moneyline (winner) markets with decimal odds derived from power rankings.

**Match events** — odds are based on the strength of each competitor:

```
strength = 7 - rank
P(home wins) = strength_home / (strength_home + strength_away)
decimal_odds = 1 / probability
```

**Performance events** — odds reflect the probability of a team winning a majority of matches (4+ out of 6). Calculated by combining all individual match probabilities. When the match count is even (regular season: 6 matches), a **Draw** selection is included for the 3-3 tie scenario. Playoffs (7 matches) have no draw.

Example match odds:

| Matchup | Home Odds | Away Odds |
|---------|-----------|-----------|
| Rank 1 vs Rank 6 | 1.17 | 7.00 |
| Rank 2 vs Rank 4 | 1.60 | 2.67 |
| Rank 3 vs Rank 3 | 2.00 | 2.00 |

Every WJAL selection carries `odds_source: "wjal"` with `odds_source_kind: "book"`.

---

## Example: Performance Event

```json
{
  "id": "de72ff88-...",
  "name": "Chargers vs Devils - 2026-02-13",
  "start_date": "2026-02-13T19:00:00Z",
  "status": "completed",
  "event_type": "performance",
  "competitors": [
    { "entity": { "uri": "wjal-chargers", "type": "team", "name": "Chargers" }, "role": "home" },
    { "entity": { "uri": "wjal-devils", "type": "team", "name": "Devils" }, "role": "away" }
  ],
  "markets": [
    {
      "type": "moneyline",
      "selections": [
        { "outcome": "Chargers", "odds_decimal": 1.49, "result": "won" },
        { "outcome": "Devils", "odds_decimal": 3.03, "result": "lost" },
        { "outcome": "Draw", "odds_decimal": 5.80, "result": "lost" }
      ]
    }
  ],
  "results": [
    { "entity_id": "aaa-...", "placement": 1 },
    { "entity_id": "bbb-...", "placement": 2 }
  ],
  "child_events": [
    { "id": "b13b07f6-...", "name": "Match 1 (Doubles): Chargers vs Devils", "event_type": "match", "status": "completed" },
    { "id": "5760685d-...", "name": "Match 2 (Doubles): Chargers vs Devils", "event_type": "match", "status": "completed" }
  ]
}
```

## Example: Match Event (Doubles)

```json
{
  "id": "b13b07f6-...",
  "name": "Match 1 (Doubles): Chargers vs Devils",
  "event_type": "match",
  "parent_event_id": "de72ff88-...",
  "competitors": [
    {
      "entity": {
        "uri": "wjal-pair-iturbide-44-ubilla-49",
        "type": "pair",
        "name": "Iturbide & Ubilla",
        "members": [
          { "entity": { "uri": "wjal-iturbide-44", "type": "player", "name": "Iturbide" } },
          { "entity": { "uri": "wjal-ubilla-49", "type": "player", "name": "Ubilla" } }
        ]
      },
      "role": "home"
    },
    {
      "entity": {
        "uri": "wjal-pair-benny-30-etcheberry-63",
        "type": "pair",
        "name": "Benny & Etcheberry",
        "members": [
          { "entity": { "uri": "wjal-benny-30", "type": "player", "name": "Benny" } },
          { "entity": { "uri": "wjal-etcheberry-63", "type": "player", "name": "Etcheberry" } }
        ]
      },
      "role": "away"
    }
  ],
  "markets": [
    {
      "type": "moneyline",
      "selections": [
        { "outcome": "Iturbide & Ubilla", "odds_decimal": 1.75, "result": "won" },
        { "outcome": "Benny & Etcheberry", "odds_decimal": 2.33, "result": "lost" }
      ]
    }
  ],
  "results": [
    { "entity_id": "ccc-...", "placement": 1 },
    { "entity_id": "ddd-...", "placement": 2 }
  ],
  "metadata": { "match_number": 1, "match_type": "D" }
}
```

---

## Results

Completed events include a `results` array with placement data for each competitor:

| Field | Type | Description |
|-------|------|-------------|
| `entity_id` | string | UUID of the competitor entity |
| `placement` | integer | 1 = winner, 2 = loser |

**Match events:** The winning player/pair gets `placement=1`, the loser gets `placement=2`.

**Performance events:** The team with more match wins gets `placement=1`. If both teams win equal matches (e.g. 3-3), both get `placement=1` (draw).

**A match can have three competitors.** Where a player or pair was substituted, the replaced
side stays in the `competitors` array, the moneyline carries the extra outcome as a losing
selection, and the placements read `1, 2, 2`. Read the market and the results rather than
assuming two competitors.

A `completed` event normally carries results, but not always — a small number are marked
completed with no `results` and no settled selections. Check for the array rather than
inferring it from the status.

---

## Jai Alai Basics

- Best of 3 sets, first to 6 points per set
- Singles (1v1) or Doubles (2v2)
- Game day: two teams, 6-7 matches
- Team with more match wins takes the game day (3-3 tie possible in regular season)

**Schedule:** typically Tue/Wed/Thu afternoon and Fri evening US Eastern at JAM Arena,
Miami, with occasional exceptions. Matches run sequentially — each starts ~5 minutes after
the previous one ends. Read `start_date`, which is UTC, rather than assuming a slot.

**Seasons:** two per year, Spring and Fall.
