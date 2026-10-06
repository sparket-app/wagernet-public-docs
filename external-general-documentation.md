# WagerNet API Guide

WagerNet is a betting data aggregation service. It collects events and odds from multiple sports providers and serves them through a unified REST API.

**Production API:** `https://bf3zb3ipuy.us-east-1.awsapprunner.com`

**OpenAPI spec:** `GET /api/openapi.yaml` | **Swagger UI:** `/swagger.html`

All endpoints in this guide are public read endpoints — no API key is needed.

All timestamps are in UTC using RFC 3339 format (e.g. `2025-11-19T17:00:00Z`).

---

## Quick Start

```bash
BASE=https://bf3zb3ipuy.us-east-1.awsapprunner.com

# What's available
curl "$BASE/api/v1/disciplines"

# Upcoming tennis matches with their current odds, 100 per page
curl "$BASE/api/v1/events?discipline_uri=tennis&status=upcoming&pageSize=100"

# One event in full
curl "$BASE/api/v1/events/{id}"

# How a market's price moved over time
curl "$BASE/api/v1/markets/{marketId}/odds-history"
```

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

**Category** — the broadest grouping. `sports`, `entertainment` and `politics` carry data today; `esports`, `financial` and `other` are defined but unused.

**Discipline** — the specific sport or activity within a category. Examples: `american-football`, `baseball`, `basketball`, `ice-hockey`, `soccer`, `tennis`, `jai-alai`, `pickleball`, `auto-racing`, `us-elections`. Each discipline belongs to exactly one category. `GET /api/v1/disciplines` returns the current list.

**Competition** — a league, tournament, or recurring series within a discipline. Examples: `nfl`, `mlb`, `premier-league`, `champions-league`, `us-open-mens-singles`, `worldoutlaws`, `wjal`. Each competition has a `country`, but its form follows the source: a code for US competitions (`US`), a country name elsewhere (`Germany`, `Türkiye`), and `International` for a competition belonging to no one country. Don't parse it as an ISO code.

In tennis a competition is one **draw**, not one tournament, and the list grows through the season as draws are published — so discover competitions from the API rather than hardcoding them.

**Season** — a time-bound instance of a competition. Examples: `mlb-2026`, `mls-2026`, `us-open-mens-singles-2026`, `wjal-spring-2026`. Has optional `start_date` and `end_date`.

**Stage** — an optional grouping within a season. Not all events have a stage. When present, it represents a phase, round, or week — for example `Regular Season`, `Quarterfinal`, `Week 1`. Stages have an optional `stage_type` (e.g. `regular-season`, `postseason`, `wild-card`, `qualifying`, `main`, `general`) and `stage_number` for ordering within the season.

### Events

An **event** is anything you can bet on — a game, a match, a race, a performance. Events always belong to a season, and optionally to a stage within that season.

Each event has an `event_type` field describing what it is. The values depend on the discipline — team sports and tennis use `match`, elections use `election`, ranking markets use `ranking`, and jai-alai and pickleball use `performance` (a game day) alongside `match` (one match within it). It can also be absent, as it is on placeholder playoff games that may never be played.

`start_date` can be absent too, on an event whose slot exists before it is scheduled. Where it is present, treat it as the scheduled time rather than a firm one: in sports played back-to-back on the same court, every match of an unscheduled round can share a single placeholder timestamp.

**Nested events:** Events can form parent-child relationships. A parent event contains `child_events[]` (lightweight summaries), and each child event has a `parent_event_id` pointing back. For example, a jai-alai game day is a parent event containing 6 individual matches as children. Not all disciplines use nesting.

Events that came from an upstream provider carry that provider's event id — `espn_id` or `sportsdataio_id` — so you can join them against another feed.

### Entities

An **entity** is an abstract participant that can appear across many events and seasons. Entity types:

| Type | Description | Example |
|------|-------------|---------|
| `team` | An organization or squad | Devils, Kansas City Chiefs |
| `player` | An individual competitor | Iturbide, Ben Shelton |
| `pair` | Two players competing together | Iturbide & Ubilla |
| `party` | A political party | Democratic Party |
| `candidate` | A person standing for office | Ro Khanna |
| `participant` | A generic participant (when type is unclear) | — |

Each entity has a globally unique `uri` that stays stable across re-imports. Most are the normalised name (e.g. `los-angeles-dodgers`, `ben-shelton`); some carry a source prefix (e.g. `wjal-devils`, `wjal-iturbide-44`).

Entities also have an optional `discipline_uri` field that indicates which discipline they belong to. This is used to disambiguate entities that might share a name across sports (e.g. a team called "Giants" exists in both NFL and MLB). When present, `discipline_uri` ties the entity to a specific discipline like `baseball` or `american-football`.

**Provider ids:** `provider_ids` maps each upstream provider to its id for the entity — for example `{ "espn": "19", "sportsdataio": "1" }` for the Los Angeles Dodgers. The keys are provider namespaces, and a provider that keys entities per sport appears once per sport: `espn-tennis` for a tennis player, `espn-soccer` for a football club. Read the keys present rather than assuming a fixed set; the map is empty for entities no upstream provider identifies.

**Composed entities:** Entities of type `pair` include a `members` array containing their individual players. This nesting lets you see both the pair as a unit and the individual players within it.

**Season rosters** associate entities with seasons. A team's roster for a given season lists its players and pairs. Pairs in the roster also include their player members nested inside.

### Competitors

A **competitor** is an entity participating in a specific event. Each competitor has:
- `entity` — the entity object (team, player, pair, party or candidate)
- `role` — their position in the event, typically `home` or `away`. Where home/away doesn't apply — a race, an election, a multi-way field — the role is `competitor` or `participant`.

The entity type of competitors varies by event type. For example, a game-day event might have `team` competitors, while an individual match might have `pair` or `player` competitors. See discipline-specific docs for details.

A two-way event can carry **more than two competitor rows** — a replaced participant stays in the array. Read the market's selections and the event's `results` rather than assuming the array's length.

### Markets

A **market** is a betting category on an event. Market types:

| Type | Description |
|------|-------------|
| `moneyline` | Who wins the event |
| `spread` | Point difference (handicap) |
| `totals` | Over/under on total points |
| `game_spread` | Games difference (handicap), where `spread` counts a larger unit — sets, in tennis |
| `game_totals` | Over/under on total games |
| `to_advance` | Which side progresses, decided including extra time and penalties |

Where a draw is possible — soccer, for example — the moneyline has a third `Draw` selection. A selection does not carry the entity it belongs to, so match a selection to a competitor by its `outcome`, which is the competitor's display name (or `Over` / `Under` / `Draw`).

Not all events have markets. Some data sources provide events without odds, in which case `markets` will be an empty array.

**A market can exist before it is priced.** The moneyline is often created with the event, and its selections then carry **no** `odds_decimal`, `odds_source` or `odds_source_kind` at all — the fields are omitted, not null. Around a fifth of upcoming soccer and college-football moneylines are in this state at any time, because books post those lines only days ahead. Treat "the market exists" and "the market has a price" as two different states, and expect missing keys rather than nulls. Spread and totals markets are usually absent entirely until priced, so the number of markets on an event varies.

### Selections

A **selection** is a single betting option within a market. Each selection includes:

| Field | Description |
|-------|-------------|
| `id` | UUID |
| `outcome` | What you're betting on (e.g. team name, "Over", "Under") |
| `odds_decimal` | Odds in decimal format (e.g. 1.95, 2.50). Absent until the market is priced. |
| `point` | The line for spread and totals markets (e.g. -3.5, 45.5). Absent for moneyline. |
| `result` | Resolution result after event completion: `won`, `lost`, `void`, `push`. Absent when unresolved. |
| `is_current` | `false` for a selection no longer offered, such as a withdrawn candidate |
| `odds_source` | Where the current odds came from (e.g. `espn-draftkings`, `kalshi`, `sparket.ai`, `tennis-model`) |
| `odds_source_kind` | What that source's prices are: `book`, `exchange` or `model`. Absent for a source not classified. |
| `updated_at` | When these odds were last refreshed |

All odds are **decimal format only**. To convert: implied probability = 1 / odds_decimal.

**Book prices are published fair, with the margin removed.** A sportsbook's posted line carries its margin, so the implied probabilities of a market sum to more than 1. WagerNet normalises them to sum to exactly 1 and publishes that fair price as `odds_decimal`, so it will not match the book's screen price. The price as posted survives only in odds history, as `odds_decimal_raw`. Removing the margin again is harmless — the operation repeats without effect — but expect a fair price, not a book one. Model prices are treated as already fair and are neither adjusted nor given a raw counterpart.

By default only current selections are returned. Pass `include_historical=true` to also get the ones no longer offered. A market whose line moved keeps the superseded selections this way, so one market can hold a whole ladder of lines with only the live one current — and a market can come back with an empty `selections` array when none of its rows is current.

### Where a price comes from

Every price carries its true origin in `odds_source`, and what kind of price that
is in `odds_source_kind`:

| Kind | What it means |
|------|---------------|
| `book` | A line a sportsbook posted and takes bets at |
| `exchange` | A traded or pooled price — what the money itself says |
| `model` | A computed price. Published where no market prices the event at all, and includes a model fused with market prices, since nobody posted it as a line |

A model price is not interchangeable with a posted one, so weigh the two apart.
A source WagerNet does not classify carries no `odds_source_kind` at all rather
than a guessed one, and some older priced selections carry no `odds_source`
either — a price without a stated origin is not evidence of one.

Markets on one event can come from different sources — a tennis match can carry a `model` moneyline next to a spread from another source — so read the source per selection, not per event.

**A selection's `odds_source` is whoever wrote its current price, not the market's history.** Model prices pushed in by a partner pipeline take precedence permanently: once such a price owns a selection, an incoming book price never replaces it, however recent. The book's observations keep landing in odds history regardless, so a market can read entirely `model` on its selections while its history is largely `book`. To follow one source over time, filter odds history by `source` rather than reading the selection.

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
GET /api/v1/events                  → events, filtered and paged
GET /api/v1/events/{id}             → single event with full details
```

**Query parameters for `/events`:**

| Parameter | Description | Example |
|-----------|-------------|---------|
| `discipline_uri` | Filter by discipline URI | `tennis` |
| `competition_uri` | Filter by competition URI | `premier-league` |
| `season_uri` | Filter by season URI | `mlb-2026` |
| `stage_uri` | Filter by stage URI | `mlb-2026-regular-season` |
| `type` | Filter by event type | `match` |
| `status` | Filter by status: `upcoming`, `live`, `completed`, `cancelled` or `postponed` | `upcoming` |
| `name` | Exact event name | — |
| `name_like` | Partial event name, case-insensitive | `Dodgers` |
| `start_date_from` | Events starting from this time (RFC 3339) | `2026-09-10T00:00:00Z` |
| `start_date_to` | Events starting before this time (RFC 3339) | `2026-09-17T00:00:00Z` |
| `include_historical` | Also return selections no longer offered | `true` |
| `sort` | `field:direction`, comma-separated. Fields: `start_date`, `end_date`, `name`, `status` | `start_date:asc` |
| `page` | Page number, starting at 1 | `2` |
| `pageSize` | Events per page, 1–100. Default 20 | `100` |

**Parameter names must match exactly.** An unrecognised parameter is ignored rather than rejected, so `?discipline=tennis` silently returns every event. Dates without a time and zone (`2026-09-10`) are rejected.

`GET /api/v1/events/{id}` also accepts `include_historical`.

The list endpoint returns both parent and child events. Parent events include `child_events[]` as lightweight summaries (id, name, event_type, start_date, status). Fetch a single event by ID for the full object including competitors with nested entity members.

**Pagination:** the list is always paged — 20 events per page unless you set `pageSize`. The response carries a `pagination` object alongside the events:

```json
{
  "events": [ ... ],
  "pagination": { "page": 1, "pageSize": 20, "total": 339, "total_pages": 17 }
}
```

Request pages until `page` reaches `total_pages`.

### Entities

```
GET /api/v1/seasons/{uri}/entities  → team rosters for a season
GET /api/v1/entities/{id}           → single entity
```

The roster endpoint returns teams with their members (players and pairs) for that season, as `{ "entities": [...] }`. Pair entities include their individual player members nested inside. It is built from whoever has appeared in an imported event rather than from a published squad list, so it grows through a season and is empty for disciplines that field individuals rather than teams — tennis and elections return an empty array.

The single-entity endpoint returns a reduced view: `id`, `uri`, `type` and `name` only, with `members` for a pair. It carries neither `discipline_uri` nor `provider_ids` — for those, read the entity embedded in an event's `competitors`.

### Odds History

```
GET /api/v1/markets/{marketId}/odds-history
```

A selection carries only its current price. This endpoint returns the append-only history of a market — every recorded price across all of its selections, newest first.

| Parameter | Description | Example |
|-----------|-------------|---------|
| `source` | Only quotes from this `odds_source` | `espn-draftkings` |
| `point` | Only quotes for this line | `-1.5` |
| `limit` | Maximum quotes to return. Default and maximum 500 | `50` |

Response format: `{ "market_id": "...", "quotes": [...] }` — see OddsQuote below.

### Reading Odds in Real Time

There is no streaming or webhook feed; poll the REST endpoints.

- Narrow the request to what you need — `discipline_uri`, `status=upcoming` or `status=live`, a `start_date_from` / `start_date_to` window — and use `pageSize=100` to keep the number of calls down.
- Each source refreshes on its own schedule, from under a minute for live games to many hours for season schedules. Polling more often than about once a minute gains little.
- Compare each selection's `updated_at` between polls to see which prices moved.
- To see every move between two polls, read the market's odds history.

---

## Object Reference

### Event

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `id` | string | yes | UUID |
| `name` | string | yes | Display name |
| `start_date` | string | no | UTC timestamp (RFC 3339) |
| `end_date` | string | no | UTC timestamp (RFC 3339) |
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
| `metadata` | object | no | Provider-specific fields, varies by discipline (e.g. `best_of` on tennis matches: 3 or 5 sets) |
| `espn_id` | string | no | ESPN's id for this event |
| `sportsdataio_id` | string | no | SportsDataIO's id for this event |

### Event Status Values

| Status | Description |
|--------|-------------|
| `upcoming` | Not started yet |
| `live` | Currently in progress |
| `completed` | Finished |
| `cancelled` | Will not take place |
| `postponed` | Delayed to a later time |
| `suspended` | Started but halted mid-game (e.g. MLB suspended games that resume later) |

`suspended` appears on events but is not accepted by the `status` filter.

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
| `uri` | string | Stable identifier (e.g. `mlb-2026`) |
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
| `stage_type` | string | Optional (e.g. `regular-season`, `wild-card`, `general`) |

### Entity

| Field | Type | Description |
|-------|------|-------------|
| `id` | string | UUID |
| `uri` | string | Globally unique, stable identifier |
| `type` | string | `team`, `player`, `pair`, `party`, `candidate`, or `participant` |
| `discipline_uri` | string | Optional — discipline this entity belongs to (e.g. `baseball`, `american-football`). Used to disambiguate entities across sports. |
| `name` | string | Display name |
| `provider_ids` | object | Optional — upstream provider → that provider's id for this entity |
| `members` | EntityMember[] | Only for composed entities (pairs) — the individual players |

### Competitor

| Field | Type | Description |
|-------|------|-------------|
| `entity` | Entity | The competing entity |
| `role` | string | Optional — `home`, `away`, `competitor`, or `participant` |

### Market

| Field | Type | Description |
|-------|------|-------------|
| `id` | string | UUID |
| `type` | string | `moneyline`, `spread`, `totals`, `game_spread`, or `game_totals` |
| `selections` | Selection[] | The betting options |

### Selection

| Field | Type | Description |
|-------|------|-------------|
| `id` | string | UUID |
| `outcome` | string | What you're betting on — a competitor's display name, or `Over` / `Under` / `Draw` |
| `odds_decimal` | number | Decimal odds (e.g. 1.95), margin removed. Absent until the market is priced. |
| `point` | number | Optional — the line for spread/totals (e.g. -3.5, 45.5) |
| `result` | string | Optional — resolution result: `won`, `lost`, `void`, or `push`. Null when unresolved. |
| `is_current` | boolean | `false` for a selection no longer offered |
| `odds_source` | string | Optional — true origin of the current odds |
| `odds_source_kind` | string | Optional — `book`, `exchange` or `model`. Absent for an unclassified source. |
| `updated_at` | string | Optional, when odds were last updated |

### OddsQuote

| Field | Type | Description |
|-------|------|-------------|
| `selection_id` | string | Selection this price belongs to |
| `outcome` | string | Outcome of that selection |
| `odds_decimal` | number | Decimal odds recorded, margin removed |
| `odds_decimal_raw` | number | Optional — the price as the book posted it, margin included. Book sources only |
| `point` | number | Optional — the line at the time, for spread/totals |
| `odds_source` | string | True origin of the price |
| `odds_source_kind` | string | Optional — `book`, `exchange` or `model` |
| `confidence` | number | Optional — model confidence in [0, 1], set only on quotes pushed by an external model pipeline |
| `run_id` | string | Optional — that pipeline's run identifier |
| `created_at` | string | When the price was recorded |

### Pagination

| Field | Type | Description |
|-------|------|-------------|
| `page` | integer | Current page, starting at 1 |
| `pageSize` | integer | Events per page |
| `total` | integer | Events matching the filters, across all pages |
| `total_pages` | integer | Number of pages |

### EventResultPlacement

| Field | Type | Description |
|-------|------|-------------|
| `entity_id` | string | UUID of the competitor entity |
| `placement` | integer | Finishing position (1 = winner). Multiple entities at placement 1 = draw/tie. |
| `score` | object | Optional — the final score, shaped by the sport |

Present only on completed events. The `results` array is ordered by placement ascending.

`score` is whatever that sport counts: `{ "points": 24 }` in American football, `{ "goals": 2 }` in soccer, and in tennis a per-set breakdown plus `sets_won` and `games_won`. Read the keys the discipline documents rather than assuming a shape.

**A draw puts every tied side at `placement: 1`.** Code that reads "placement 1 = the winner" will report two winners on a drawn match. How the moneyline then settles depends on whether the market offers a draw: where a `Draw` selection exists it wins and both sides lose, and where none exists every side pushes.

---

## Discipline-Specific Docs

- [American Football](external-american-football-documentation.md) — NFL and college football
- [Soccer](external-soccer-documentation.md)
- [Tennis](external-tennis-documentation.md)
- [MLB (Baseball)](external-mlb-documentation.md)
- [WJAL (Jai-Alai)](external-wjal-documentation.md)
- [US Elections](external-elections-documentation.md)

Basketball, ice hockey, pickleball, auto-racing and Netflix ranking markets are also served but have no
page of their own yet; discover them through `GET /api/v1/disciplines`.
