# WJAL Data Model in WagerNet

How World Jai-Alai League (Battle Court at JAM Arena, Miami) data maps into WagerNet.

See [General Documentation](external-general-documentation.md) for general WagerNet API documentation.

---

## Data Hierarchy

```
Discipline: jai-alai
└── Competition: wjal
    └── Season: wjal-spring-2026
        ├── Stage: Regular Season
        ├── Stage: Playoffs
        └── Stage: Championship
            └── Performance (parent event, event_type=performance)
                ├── Match 1 (Doubles) (child event, event_type=match)
                ├── Match 2 (Doubles)
                ├── ...
                └── Match 6 (Doubles)
```

- **Stage** = season phase (Regular Season, Playoffs, Championship)
- **Performance** = game day (two teams, 6-7 matches)
- **Match** = individual match (child event linked via `parent_event_id`)

---

## Quick Start

```bash
# All jai-alai events
curl "https://bf3zb3ipuy.us-east-1.awsapprunner.com/api/v1/events?discipline=jai-alai"

# Season stages
curl "https://bf3zb3ipuy.us-east-1.awsapprunner.com/api/v1/seasons/wjal-spring-2026/stages"

# Team rosters (players + pairs)
curl "https://bf3zb3ipuy.us-east-1.awsapprunner.com/api/v1/seasons/wjal-spring-2026/entities"
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

| Team | URI |
|------|-----|
| Dejada Devils | `wjal-devils` |
| Wall Warriors | `wjal-warriors` |
| Chargers | `wjal-chargers` |
| Cyclones | `wjal-cyclones` |
| Fireballs | `wjal-fireballs` |
| Rebote Renegades | `wjal-renegades` |

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
| Rank 2 vs Rank 4 | 1.40 | 3.50 |
| Rank 3 vs Rank 3 | 2.00 | 2.00 |

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
        { "outcome": "Chargers", "odds_decimal": 1.49 },
        { "outcome": "Devils", "odds_decimal": 3.03 },
        { "outcome": "Draw", "odds_decimal": 5.80 }
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
        { "outcome": "Iturbide & Ubilla", "odds_decimal": 1.75 },
        { "outcome": "Benny & Etcheberry", "odds_decimal": 2.33 }
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

Event `status` is determined by results — an event is `completed` only when results are present.

---

## Jai Alai Basics

- Best of 3 sets, first to 6 points per set
- Singles (1v1) or Doubles (2v2)
- Game day: two teams, 6-7 matches
- Team with more match wins takes the game day (3-3 tie possible in regular season)

**Schedule:** Tue/Wed/Thu 3pm EST, Fri 7pm EST at JAM Arena, Miami. Matches run sequentially — each starts ~5 minutes after the previous one ends.

**Seasons:** Multiple per year (Spring, Fall, Winter)
