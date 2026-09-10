# US Elections Data Model in WagerNet

How US election data maps into WagerNet.

See [General Documentation](external-general-documentation.md) for general WagerNet API documentation.

---

## Data Hierarchy

```
Category: politics
└── Discipline: us-elections
    ├── Competition: us-house          (US House of Representatives)
    ├── Competition: us-senate         (US Senate)
    ├── Competition: us-governor       (US Governors)
    └── Competition: us-state-local    (US State and Local Elections)
        └── Season: us-state-local-2026        (one season per election cycle)
            └── Stage: us-state-local-2026-general   (stage_type: general)
                └── Event (event_type=election)
```

- **Competition** = the office being contested. A race is filed under the chamber or office it decides; mayors, city councils, state legislatures, attorneys general, secretaries of state and territorial offices all fall under `us-state-local`.
- **Season** = one election cycle, named `<competition>-<year>` (e.g. `us-house-2026`).
- **Stage** = the general election for that cycle, named `<season>-general`.
- **Event** = one race.

Election events are flat — no `parent_event_id`, no `child_events`.

### Election dates

The stage carries the election date in its `start_date`, and every event in that stage inherits it. `start_date` is **the moment polls open** — midnight Eastern on election day — and `end_date` is the following midnight, so the window covers election night, not the date a winner is sworn in.

This date does not come from the odds source. No provider that publishes election prices also publishes the date of the election, so WagerNet keeps its own election calendar and stamps it onto the events. Treat `start_date` as authoritative for "when is this race decided"; do not try to derive it from anything else in the payload.

---

## Quick Start

```bash
# All US election events (paged, pageSize max 100)
curl "https://bf3zb3ipuy.us-east-1.awsapprunner.com/api/v1/events?discipline_uri=us-elections&pageSize=100"

# One competition's races
curl "https://bf3zb3ipuy.us-east-1.awsapprunner.com/api/v1/events?competition_uri=us-senate"

# Season stages (the election date lives here)
curl "https://bf3zb3ipuy.us-east-1.awsapprunner.com/api/v1/seasons/us-house-2026/stages"

# One event, including candidates no longer in the race
curl "https://bf3zb3ipuy.us-east-1.awsapprunner.com/api/v1/events/{id}?include_historical=true"

# Price history for a market
curl "https://bf3zb3ipuy.us-east-1.awsapprunner.com/api/v1/markets/{marketId}/odds-history?source=kalshi&limit=50"
```

Use `discipline_uri`, not `discipline` — an unrecognised filter name is ignored and you get every discipline back.

---

## Scope

**Covered:** races for an office decided at the general election of one cycle — US House and Senate seats, governors and lieutenant governors, state and territorial offices, state legislative seats, mayors, city councils, judicial races, plus the two chamber-control markets ("Which party will win the U.S. House?" / "…the U.S. Senate?").

**Not covered:** primaries and party nominations, margin-of-victory and turnout ladders, seat-count brackets, ballot measures and constitutional amendments, multi-leg combination markets, and races belonging to any other election cycle.

**A handful of events sit outside that rule.** The feed also carries a few special
elections, a first-round market, and one market on which candidate will lead an opinion
poll — and all of them are filed under the cycle's general-election stage like everything
else. Nothing in the data marks them apart, so a consumer that assumes every event is a
general-election race for an office will mis-file them. `metadata.kalshi_series_ticker` is
the practical signal.

The feed holds several hundred races, most of them `us-house`, and the counts move as races
are added, redrawn and settled. Read them from `pagination.total` rather than assuming a
size.

---

## Events

Every election event has `event_type: "election"` and exactly **one market**, of type `moneyline`. The market has **one selection per contested outcome**, and the field is mutually exclusive: exactly one selection wins.

The outcomes are of one of two kinds:

| Outcome kind | Selection `outcome` | Competitor entity type |
|---|---|---|
| The parties contesting the seat | `"Democratic Party"`, `"Republican Party"` | `party` |
| The people running | `"Ro Khanna"`, `"Xavier Becerra"` | `candidate` |

**Which kind a race uses follows the source contract and changes over time.** A race listed
with party outcomes can be relisted with the candidates named instead once they are known,
and most House races now name people. Don't assume a race keeps the shape you first saw it
in.

**The `competitors` array and the selections can disagree.** A race relisted with candidate
outcomes keeps the party entities in `competitors`, so an event can list four competitors —
two parties and two people — while its market offers only the two people. Read the market's
selections for what is actually offered, and use `competitors` only as the set of entities
that have ever been attached to the race.

Where a notable independent stands against the two major parties, the field is the two
parties plus that person — for example Rhode Island's governor race lists Democratic Party,
Republican Party and Ken Block. This shape is rare.

---

## Competitors and Entities

Every competitor has `role: "competitor"` — there is no home or away in an election.

Entities are of type `party` or `candidate`, and both carry `discipline_uri: "us-elections"`. Entity URIs are the name normalised — `democratic-party`, `republican-party`, `ro-khanna`. A person is one entity across every race they appear in, so a candidate standing for two offices resolves to the same `id` and `uri` in both.

There are exactly two party entities. Candidate entities grow with the field and now number
in the thousands.

Elections have no season roster — `GET /api/v1/seasons/{uri}/entities` returns an empty array. Read the participants off each event's `competitors`.

---

## Odds

Odds are **decimal only**, in `odds_decimal`, and every selection carries
`odds_source: "kalshi"` with `odds_source_kind: "exchange"` — a traded price, not a
bookmaker's line. Because it is already a traded price, no margin is removed from it.

These prices do not come from a bookmaker. They originate on a regulated event exchange, where each outcome is a contract that pays a fixed amount if it happens and nothing if it does not — so the traded price **is** the market's implied probability, and the decimal odds published here are its reciprocal:

```
implied_probability = 1 / odds_decimal
```

There is no line to price against, so election selections carry **no `point`** — no spread, no total, no handicap. `moneyline` is the only market type.

Two consequences worth designing for:

- **A field does not sum to exactly 1.** Each selection is quoted independently and carries its own `updated_at`, so a field is not a synchronised snapshot, and the exchange's own spread sits on top of that. Measured on 2026-09-10 the implied probabilities of a field summed to a median of 1.00, but the spread is wide — from 0.55 to 1.40. Raw market probabilities are published as they are; normalise the field yourself if you need them to add to 1, and don't size your tolerance off the median.
- **Long shots produce very large decimal odds.** Values reach 1000.0, from a contract quoted at a 0.1% chance.

### Odds History

Selections carry only their current price. The append-only history of a market is a separate endpoint:

```
GET /api/v1/markets/{marketId}/odds-history?source=kalshi&limit=50
```

A row is appended **when a price moves**, not on a fixed schedule — a race whose quotes are unchanged across many sync passes adds no rows. Quotes come back newest first, interleaved across the market's selections, each tagged with its `selection_id` and `odds_source`:

```json
{
  "market_id": "5e6628a2-be5a-4764-87b4-eba951f64613",
  "quotes": [
    { "selection_id": "0393fcae-...", "outcome": "Brandon Herrera", "odds_source": "kalshi",
      "odds_source_kind": "exchange", "odds_decimal": 1.4599,
      "created_at": "2026-09-09T19:24:49.822474Z" },
    { "selection_id": "b08a22e7-...", "outcome": "Katy Padilla Stout", "odds_source": "kalshi",
      "odds_source_kind": "exchange", "odds_decimal": 3.5088,
      "created_at": "2026-09-08T19:24:43.807208Z" },
    { "selection_id": "0393fcae-...", "outcome": "Brandon Herrera", "odds_source": "kalshi",
      "odds_source_kind": "exchange", "odds_decimal": 1.3889,
      "created_at": "2026-09-07T19:25:05.414464Z" }
  ]
}
```

**`outcome` is the selection's label as it stands now, not as it stood when the quote was
recorded.** The source can relabel a contract in place — the same selection id that once
read `"Democratic Party"` can read a candidate's name today — and the whole history then
comes back under the new label. Join on `selection_id` if you need a stable series.

`source` is optional; omit it to see quotes from every source on the market. `limit` defaults
to 500 and is capped there. A `point` filter also exists, but elections carry no lines, so it
has nothing to select on here.

---

## Selections and `is_current`

A candidate who leaves the race — eliminated, disqualified, never on the ballot — is **marked `is_current: false`, not deleted**. The row keeps its `id`, so a consumer holding a selection id can always resolve it, and anything built over the full original field still settles.

By default the API returns only current selections. Pass `include_historical=true` on `/events` or `/events/{id}` to get the rest:

```bash
# 2 selections — the field still standing
curl ".../api/v1/events/89939329-f326-4d18-818a-d68689dab078"

# 25 selections — every candidate who ever appeared
curl ".../api/v1/events/89939329-f326-4d18-818a-d68689dab078?include_historical=true"
```

Two things follow from this:

- **The default view of an event can have an empty `selections` array** while the event is still `upcoming`. This happens where the outcome became certain before the vote — some races are decided by the shape of the ballot months ahead. Nothing is offered on them, so no selection is current, but the market object is still there and the historical view shows the settled prices.
- **Once the race itself is called, the whole field is current again**, as the final board.

---

## Event Metadata

Election events carry three `metadata` keys, all provenance from the source exchange:

| Key | Type | Meaning |
|---|---|---|
| `kalshi_series_ticker` | string | The source's identifier for the template the race came from. Races generated from one template share it — 358 House district races carried `KXHOUSERACE` on 2026-07-30 — while one-off races have their own. It is not a competition and not a race id. |
| `kalshi_sub_title` | string | The source's short label for the race, e.g. `"TX-23"` for a district race or `"In 2026"` for a statewide one. Useful for display, not reliable as a key. |
| `mutually_exclusive` | boolean | Whether exactly one outcome of the field can win. `true` on every imported election event — only exclusive fields are imported. |

---

## Event Status

Election events use the standard statuses. What drives them here is the election calendar plus whether the race has been called:

| Status | Meaning |
|---|---|
| `upcoming` | Election day has not arrived. This holds even where the outcome is already certain. |
| `live` | Election day has arrived and no winner has been declared. Trading continues through the night; there is no in-play concept. |
| `completed` | A winner has been declared. |

An event never reports `completed` before its `start_date`, whatever the prices say. A settled contract on an `upcoming` event means that outcome is out of the running, not that the race is over.

On 2026-09-10 every election event was `upcoming` — the 2026 general election had not been held.

---

## Results

When a race is called, the event moves to `completed` and gains a `results` array, one entry per outcome:

| Field | Type | Description |
|---|---|---|
| `entity_id` | string | UUID of the party or candidate entity |
| `placement` | integer | `1` for the winner, `2` for everyone else |

Elections have exactly one winner and no draw, so exactly one entry carries `placement: 1`, and there is no push case. Selections on the moneyline gain a `result` of `won` for the winner and `lost` for the rest. Since elections have no score, no score fields are populated.

---

## One Race Can Be Two Events

**This is the caveat most likely to bite a consumer.** The same election can be listed by the source both as a contract on which **party** takes the seat and as a contract on which **person** takes it. WagerNet publishes one event per source contract, so that election appears **twice**, and nothing in the data links the two.

CA-17 on 2026-07-30:

| Event id | Name | `kalshi_series_ticker` | Outcomes |
|---|---|---|---|
| `8840558b-7dd1-4190-ab4e-7c6597965b4f` | CA-17 House winner? | `KXHOUSERACE` | Ro Khanna, Ritesh Tandon |
| `74705d78-295b-40ec-bbfb-9ac50db08394` | CA-17 winner? (Person) | `KXCA17PERSON` | Ro Khanna, Nicholas Finan, Ethan Agarwal, Ha T Phan, Ritesh Tandon, Jason Park |

Same seat, same `competition_uri`, same `start_date`, different ids, unrelated markets — and
here both now name people, so the two events are not even told apart by the kind of outcome
they offer. California Governor is the same story: `California Governor winner?` alongside
`California Governor winner? (Person)`. Some races carry an explicit `(Party)` or `(Person)`
suffix in the `name`, but most do not.

The reason is that the source publishes no identifier for the underlying race — only for each contract on it. There is no field to join on, and WagerNet does not invent one. A consumer that counts events, aggregates volume, or builds one product per event will **double count** these races unless it deduplicates itself; matching on `competition_uri` plus the district or office in `name` / `kalshi_sub_title` is the practical approach, and it is heuristic.

---

## Example: A House Race

Filed as a party race by the source and later relabelled with the candidates, so its
`competitors` hold both parties and both people while the market offers only the people.

```json
{
  "id": "010f8017-4bc6-4a36-9735-8a5086a64f2e",
  "name": "TX-23 House winner?",
  "start_date": "2026-11-03T05:00:00Z",
  "end_date": "2026-11-04T05:00:00Z",
  "status": "upcoming",
  "event_type": "election",
  "category": { "uri": "politics", "name": "Politics" },
  "discipline": { "uri": "us-elections", "name": "US Elections", "category_uri": "politics" },
  "competition": { "uri": "us-house", "name": "US House of Representatives", "country": "US", "discipline_uri": "us-elections" },
  "season": { "uri": "us-house-2026", "name": "US House 2026", "competition_uri": "us-house" },
  "stage": { "uri": "us-house-2026-general", "name": "General Election", "stage_type": "general", "stage_number": 2,
             "start_date": "2026-11-03T05:00:00Z", "end_date": "2026-11-04T05:00:00Z" },
  "competitors": [
    { "entity": { "uri": "democratic-party", "type": "party", "discipline_uri": "us-elections", "name": "Democratic Party" }, "role": "competitor" },
    { "entity": { "uri": "republican-party", "type": "party", "discipline_uri": "us-elections", "name": "Republican Party" }, "role": "competitor" },
    { "entity": { "uri": "katy-padilla-stout", "type": "candidate", "discipline_uri": "us-elections", "name": "Katy Padilla Stout" }, "role": "competitor" },
    { "entity": { "uri": "brandon-herrera", "type": "candidate", "discipline_uri": "us-elections", "name": "Brandon Herrera" }, "role": "competitor" }
  ],
  "markets": [
    {
      "id": "5e6628a2-be5a-4764-87b4-eba951f64613",
      "type": "moneyline",
      "selections": [
        { "id": "b08a22e7-...", "outcome": "Katy Padilla Stout", "odds_decimal": 3.2787,
          "is_current": true, "odds_source": "kalshi", "odds_source_kind": "exchange",
          "updated_at": "2026-09-09T19:24:49.805820Z" },
        { "id": "0393fcae-...", "outcome": "Brandon Herrera", "odds_decimal": 1.4599,
          "is_current": true, "odds_source": "kalshi", "odds_source_kind": "exchange",
          "updated_at": "2026-09-09T19:24:49.817621Z" }
      ]
    }
  ],
  "metadata": {
    "kalshi_series_ticker": "KXHOUSERACE",
    "kalshi_sub_title": "TX-23",
    "mutually_exclusive": true
  }
}
```

## Example: Person-Outcome Race, With Withdrawn Candidates

Fetched with `include_historical=true`; the default view returns only the two current selections.

```json
{
  "id": "89939329-f326-4d18-818a-d68689dab078",
  "name": "California Governor winner? (Person)",
  "start_date": "2026-11-03T05:00:00Z",
  "status": "upcoming",
  "event_type": "election",
  "competition": { "uri": "us-governor", "name": "US Governors", "country": "US", "discipline_uri": "us-elections" },
  "competitors": [
    { "entity": { "id": "9c376bf2-...", "uri": "eleni-kounalakis", "type": "candidate", "discipline_uri": "us-elections", "name": "Eleni Kounalakis" }, "role": "competitor" },
    { "entity": { "id": "836277d6-...", "uri": "chad-bianco", "type": "candidate", "discipline_uri": "us-elections", "name": "Chad Bianco" }, "role": "competitor" }
  ],
  "markets": [
    {
      "type": "moneyline",
      "selections": [
        { "id": "e77aa125-...", "outcome": "Xavier Becerra", "odds_decimal": 1.0449,
          "is_current": true, "odds_source": "kalshi", "updated_at": "2026-07-29T22:18:36.122141Z" },
        { "id": "b112b585-...", "outcome": "Steve Hilton", "odds_decimal": 22.4719,
          "is_current": true, "odds_source": "kalshi", "updated_at": "2026-07-29T22:18:36.139152Z" },
        { "id": "0b9d9cfc-...", "outcome": "Toni Atkins", "odds_decimal": 0,
          "is_current": false, "odds_source": "kalshi", "updated_at": "2026-07-28T21:55:50.838218Z" },
        { "id": "7c305c74-...", "outcome": "Rick Caruso", "odds_decimal": 0,
          "is_current": false, "odds_source": "kalshi", "updated_at": "2026-07-28T21:55:50.850612Z" }
      ]
    }
  ],
  "metadata": {
    "kalshi_series_ticker": "KXGOVCA",
    "kalshi_sub_title": "In 2026",
    "mutually_exclusive": true
  }
}
```

The competitors array lists every entity that has appeared in the field, current or not — 25 of them on this event on 2026-07-30, trimmed to two above.
