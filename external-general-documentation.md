# WagerNet API Guide

WagerNet is a betting data aggregation service. It collects events and odds from multiple sports providers and serves them through a unified REST API.

**Production API:** `https://bf3zb3ipuy.us-east-1.awsapprunner.com`

**OpenAPI spec:** `GET /api/openapi.yaml` | **Swagger UI:** `/swagger.html`

All timestamps are in UTC using RFC 3339 format (e.g. `2025-11-19T17:00:00Z`).

---

## Data Model

WagerNet organizes sports data in a fixed hierarchy. Each level nests inside the one above it:

```
Category
└── Discipline
    └── Competition
        └── Season
            └── Stage (optional)
                └── Event
                    ├── Competitors (entities with roles)
                    ├── Child Events (optional nesting)
                    └── Market
                        └── Selection
```

### Hierarchy Levels

**Category** — the broadest grouping. Values: `sports`, `entertainment`, `politics`, `esports`, `financial`.

**Discipline** — the specific sport or activity within a category. Examples: `american-football`, `auto-racing`, `jai-alai`, `baseball`. Each discipline belongs to exactly one category.

**Competition** — a league, tournament, or recurring series within a discipline. Examples: `nfl`, `world-of-outlaws`, `wjal`, `mlb`. Each competition has a `country` field.

**Season** — a time-bound instance of a competition. Examples: `nfl-2025`, `wjal-fall-2025`, `mlb-2026`. Has optional `start_date` and `end_date`.

**Stage** — an optional grouping within a season. Not all events have a stage. When present, it represents a phase, round, or week — for example `Regular Season`, `Playoffs`, `Week 1`. Stages have an optional `stage_type` (e.g. `regular`, `playoffs`, `championship`) and `stage_number` for ordering.

### Events

An **event** is anything you can bet on — a game, a match, a race, a performance. Events always belong to a season, and optionally to a stage within that season.

Each event has an `event_type` field describing what it is. The values depend on the discipline — for example, jai-alai uses `performance` (game day) and `match` (individual match), while other sports might use `game` or `race`.

**Nested events:** Events can form parent-child relationships. A parent event contains `child_events[]` (lightweight summaries), and each child event has a `parent_event_id` pointing back. For example, a jai-alai game day is a parent event containing 6 individual matches as children. Not all disciplines use nesting.

### Entities

An **entity** is an abstract participant that can appear across many events and seasons. Entity types:

| Type | Description | Example |
|------|-------------|---------|
| `team` | An organization or squad | Devils, Kansas City Chiefs |
| `player` | An individual competitor | Iturbide, Bradley |
| `pair` | Two players competing together | Iturbide & Ubilla |
| `participant` | A generic participant (when type is unclear) | — |

Each entity has a globally unique `uri` that stays stable across re-imports. URIs include a provider prefix to avoid collisions across disciplines (e.g. `wjal-devils`, `wjal-iturbide-44`).

Entities also have an optional `discipline_uri` field that indicates which discipline they belong to. This is used to disambiguate entities that might share a name across sports (e.g. a team called "Giants" exists in both NFL and MLB). When present, `discipline_uri` ties the entity to a specific discipline like `baseball` or `american-football`.

**Composed entities:** Entities of type `pair` include a `members` array containing their individual players. This nesting lets you see both the pair as a unit and the individual players within it.

**Season rosters** associate entities with seasons. A team's roster for a given season lists its players and pairs. Pairs in the roster also include their player members nested inside.

### Competitors

A **competitor** is an entity participating in a specific event. Each competitor has:
- `entity` — the entity object (team, player, or pair)
- `role` — their position in the event, typically `home` or `away`. Can also be `participant` when home/away doesn't apply.

The entity type of competitors varies by event type. For example, a game-day event might have `team` competitors, while an individual match might have `pair` or `player` competitors. See discipline-specific docs for details.

### Markets

A **market** is a betting category on an event. Market types:

| Type | Description |
|------|-------------|
| `moneyline` | Who wins the event |
| `spread` | Point difference (handicap) |
| `totals` | Over/under on total points |

Not all events have markets. Some data sources provide events without odds, in which case `markets` will be an empty array.

### Selections

A **selection** is a single betting option within a market. Each selection includes:

| Field | Description |
|-------|-------------|
| `id` | UUID |
| `outcome` | What you're betting on (e.g. team name, "Over", "Under") |
| `odds_decimal` | Odds in decimal format (e.g. 1.95, 2.50) |
| `point` | The line for spread and totals markets (e.g. -3.5, 45.5). Absent for moneyline. |
| `result` | Resolution result after event completion: `won`, `lost`, `void`, `push`. Absent when unresolved. |
| `updated_at` | When these odds were last refreshed |

All odds are **decimal format only**. To convert: implied probability = 1 / odds_decimal.

---

## API Endpoints

### Hierarchy Navigation

Walk the hierarchy top-down to discover what's available:

```
GET /api/v1/disciplines                          → list all disciplines
GET /api/v1/disciplines/{uri}/competitions       → competitions in a discipline
GET /api/v1/competitions/{uri}/seasons           → seasons in a competition
GET /api/v1/seasons/{uri}/stages                 → stages in a season
```

Each response wraps results in a named array: `{ "disciplines": [...] }`, `{ "competitions": [...] }`, `{ "seasons": [...] }`, `{ "stages": [...] }`.

All objects have a stable `uri` field used as the identifier in URL paths.

### Events

```
GET /api/v1/events                  → all events (filterable)
GET /api/v1/events/{id}             → single event with full details
```

**Query parameters for `/events`:**

| Parameter | Description | Example |
|-----------|-------------|---------|
| `discipline` | Filter by discipline URI | `jai-alai` |
| `type` | Filter by event type | `match` |
| `status` | Filter by event status | `upcoming` |
| `stage` | Filter by stage URI | `wjal-fall-2025-regular` |

The list endpoint returns both parent and child events. Parent events include `child_events[]` as lightweight summaries (id, name, event_type, start_date, status). Fetch a single event by ID for the full object including competitors with nested entity members.

Response format: `{ "events": [...] }`

### Entities (Season Rosters)

```
GET /api/v1/seasons/{uri}/entities  → team rosters for a season
```

Returns teams with their members (players and pairs) for that season. Pair entities include their individual player members nested inside.

Response format: `{ "entities": [...] }`

---

## Object Reference

### Event

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `id` | string | yes | UUID |
| `name` | string | yes | Display name |
| `start_date` | string | no | UTC timestamp (RFC 3339) |
| `status` | string | yes | See status values below |
| `event_type` | string | no | Kind of event — discipline-specific |
| `parent_event_id` | string | no | Present only on child events |
| `category` | Category | yes | Top-level category |
| `discipline` | Discipline | no | Sport / discipline |
| `competition` | Competition | yes | League / competition |
| `season` | Season | no | Season this event belongs to |
| `stage` | Stage | no | Stage within the season (not all events have one) |
| `competitors` | Competitor[] | yes | Who is competing (may be empty) |
| `child_events` | EventSummary[] | no | Sub-events if this is a parent |
| `markets` | Market[] | yes | Betting markets (may be empty) |
| `results` | EventResultPlacement[] | no | Result placements for completed events |
| `metadata` | object | no | Provider-specific fields, varies by discipline |

### Event Status Values

| Status | Description |
|--------|-------------|
| `upcoming` | Not started yet |
| `live` | Currently in progress |
| `completed` | Finished |
| `cancelled` | Will not take place |
| `postponed` | Delayed to a later time |
| `suspended` | Started but halted mid-game (e.g. MLB suspended games that resume later) |

### Category

| Field | Type | Description |
|-------|------|-------------|
| `id` | string | UUID |
| `uri` | string | Stable identifier (e.g. `sports`) |
| `name` | string | Display name |

### Discipline

| Field | Type | Description |
|-------|------|-------------|
| `id` | string | UUID |
| `uri` | string | Stable identifier (e.g. `jai-alai`) |
| `name` | string | Display name |
| `category_uri` | string | Parent category URI |

### Competition

| Field | Type | Description |
|-------|------|-------------|
| `id` | string | UUID |
| `uri` | string | Stable identifier (e.g. `wjal`) |
| `name` | string | Display name |
| `country` | string | Country code (e.g. `US`) |
| `discipline_uri` | string | Parent discipline URI |

### Season

| Field | Type | Description |
|-------|------|-------------|
| `id` | string | UUID |
| `uri` | string | Stable identifier (e.g. `wjal-fall-2025`) |
| `name` | string | Display name |
| `competition_uri` | string | Parent competition URI |
| `start_date` | string | Optional, UTC timestamp |
| `end_date` | string | Optional, UTC timestamp |

### Stage

| Field | Type | Description |
|-------|------|-------------|
| `id` | string | UUID |
| `uri` | string | Stable identifier |
| `name` | string | Display name (e.g. "Regular Season", "Week 1") |
| `season_uri` | string | Parent season URI |
| `start_date` | string | Optional, UTC timestamp |
| `end_date` | string | Optional, UTC timestamp |
| `stage_number` | integer | Optional, for ordering stages within a season |
| `stage_type` | string | Optional (e.g. `regular`, `playoffs`, `championship`) |

### Entity

| Field | Type | Description |
|-------|------|-------------|
| `id` | string | UUID |
| `uri` | string | Globally unique, stable identifier |
| `type` | string | `team`, `player`, `pair`, or `participant` |
| `discipline_uri` | string | Optional — discipline this entity belongs to (e.g. `baseball`, `american-football`). Used to disambiguate entities across sports. |
| `name` | string | Display name |
| `members` | EntityMember[] | Only for composed entities (pairs) — the individual players |

### Competitor

| Field | Type | Description |
|-------|------|-------------|
| `entity` | Entity | The competing entity |
| `role` | string | Optional — typically `home`, `away`, or `participant` |

### Market

| Field | Type | Description |
|-------|------|-------------|
| `id` | string | UUID |
| `type` | string | `moneyline`, `spread`, or `totals` |
| `selections` | Selection[] | The betting options |

### Selection

| Field | Type | Description |
|-------|------|-------------|
| `id` | string | UUID |
| `outcome` | string | What you're betting on (e.g. team name, "Over 45.5") |
| `odds_decimal` | number | Decimal odds (e.g. 1.95) |
| `point` | number | Optional — the line for spread/totals (e.g. -3.5, 45.5) |
| `result` | string | Optional — resolution result: `won`, `lost`, `void`, or `push`. Null when unresolved. |
| `updated_at` | string | Optional, when odds were last updated |

### EventResultPlacement

| Field | Type | Description |
|-------|------|-------------|
| `entity_id` | string | UUID of the competitor entity |
| `placement` | integer | Finishing position (1 = winner). Multiple entities at placement 1 = draw/tie. |

Present only on completed events. The `results` array is ordered by placement ascending.

---

## Discipline-Specific Docs

- [WJAL (Jai-Alai)](external-wjal-documentation.md)
- [MLB (Baseball)](external-mlb-documentation.md)
