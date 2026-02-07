# WJAL Data Model in WagerNet

How World Jai-Alai League (Battle Court at JAM Arena, Miami) data maps into WagerNet. No odds are provided by WJAL — events have empty `markets[]`.

See [General Documentation](external-general-documentation.md) for general WagerNet API documentation.

---

## Data Hierarchy

```
Discipline: jai-alai
└── Competition: wjal
    └── Season: wjal-fall-2025
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
curl "https://bf3zb3ipuy.us-east-1.awsapprunner.com/api/v1/seasons/wjal-fall-2025/stages"

# Team rosters (players + pairs)
curl "https://bf3zb3ipuy.us-east-1.awsapprunner.com/api/v1/seasons/wjal-fall-2025/entities"
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

## Example: Performance Event

```json
{
  "id": "de72ff88-...",
  "name": "Chargers vs Devils - 2025-11-19",
  "start_date": "2025-11-19T17:00:00Z",
  "status": "completed",
  "event_type": "performance",
  "competitors": [
    { "entity": { "uri": "wjal-chargers", "type": "team", "name": "Chargers" }, "role": "home" },
    { "entity": { "uri": "wjal-devils", "type": "team", "name": "Devils" }, "role": "away" }
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
  "metadata": { "match_number": 1, "match_type": "D" }
}
```

---

## Jai Alai Basics

- Best of 3 sets, first to 6 points per set
- Singles (1v1) or Doubles (2v2)
- Game day: two teams, 6-7 matches
- Team with more match wins takes the game day

**Schedule:** Tue/Wed/Thu 3pm EST, Fri 7pm EST at JAM Arena, Miami

**Seasons:** Multiple per year (Spring, Fall, Winter)
