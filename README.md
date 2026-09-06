# AGW Platform — Canonical Specs

This repository holds the **single source of truth** for four platform-wide
contracts: order **states, workflows, and events**, proposal **statuses and
transitions**, the **traveller and company profiles** an agency curates, and
the **identifier policy** governing every id the API returns.

Each spec is normative on its own terms. Together they define what the platform
calls things and what it is allowed to do to them.

## [ORDER-STATE-MACHINE.md](ORDER-STATE-MACHINE.md) — order lifecycle

- The **12 canonical order statuses** (`Pending`, `Issued`, terminal EXIT
  statuses, transitional `PartFlown`, and the recoverable `Blocked`/`Unknown`).
- The **Workflows → Transitions → Events** contract table — the exhaustive,
  normative list of valid `(from, to)` pairs and the event each one emits.
- The **naming convention**: workflows are `PascalCase` (`AirOrderCreate`),
  requests are `camelCase` (`airOrderCreate`). Because a workflow is named after
  its terminating request, case is the only thing that tells them apart — and it
  is normative.
- The **coupon model** used to derive the flown EXIT statuses
  (`AllFlown`, `SomeFlown`, `AllNoShow`).
- The **`airOrderChangeNotif`** inbound callback and its canonical `TYPE` enum
  (informational only — never a status change).
- The **API Requester** layer: how provider-specific request sequences collapse
  atomically into exactly one canonical transition and one event, plus the full
  workflow ↔ request catalogue.
- The **layer contract** binding persistence, controller/service, API, front
  end, and tests to the exact canonical names.

## [PROPOSALS.md](PROPOSALS.md) — proposal lifecycle

- The **7 canonical proposal statuses** (`New`, `Open`, `Sent`, `Pending`, and
  the terminal `Expired`, `Cancelled`, `Confirmed`).
- The **normative transition table** — the exhaustive list of valid
  `(from, to)` pairs, the `camelCase` operation causing each, who may cause it,
  and the history entry it writes. There is **no** PascalCase workflow layer: a
  proposal transition is one action in one kick, never a provider-specific
  request sequence.
- The **independence rule**: a proposal is a wrapper around orders, not a
  projection of them. No proposal status changes because an order changed, and
  no order status changes because a proposal changed.
- **`Pending` means two different things** — proposal `Pending` (the agency owes
  a move) and order `Pending` (held with the airline). The overlap is deliberate
  and the disambiguation rules are normative.
- **Send is its own operation.** Attaching an order does not send a proposal,
  which is what makes `Pending` → `Sent` (resend) meaningful.
- The **6 option statuses**, and why a proposal may carry **more than one**
  `Approved` option.
- **Three vocabularies meet on a proposal** and must never be mixed: the
  proposal's seven statuses, an option's six, and each backing order's own.
  `Expired` and `Pending` exist in more than one and mean different things.

## [PROFILES.md](PROFILES.md) — traveller and company profiles

- The **two items** — `Traveller` and `Company` — field by field, including the
  document shape that must match a booking passenger's exactly, and why there
  is no `nationality` field.
- The **3 traveller statuses** (`Provisional`, `Active`, `Inactive`) and the
  **2 company statuses**, with the normative transition table. `Provisional`
  is the load-bearing one: resolution may **only** mint `Provisional`, so an
  agency can always tell a roster it curated from profiles the platform minted
  from booking payloads.
- **`Active` is an invariant, not a flag** — a profile may only be `Active`
  when it carries an email, a name and a surname, the three fields without
  which it cannot become a booking passenger.
- The **identity rule**: a traveller is identified by **email**, unique within
  a **company** — not within an agency, which is why resolving an address
  across an agency is a `409` and never a guess.
- The **associations table** with cardinality and delete behaviour, and the
  **tenancy rule**: a traveller has no `agency_id` and must never gain one;
  the agency is derived through the company, scoped in SQL, and a foreign
  profile is `404`, never `403`.
- The **validations**, each with the response code it produces and the layer
  that enforces it — and why `email` is optional in the data model but
  required on create.
- **A profile is a source; a booking is a snapshot.** Neither silently
  rewrites the other.
- The **namespace decision**: profiles live under `/v2/profiles`, outside
  `air`, because no airline is involved in creating one.

## [IDS.md](IDS.md) — the AGW ID policy

- The rule: **mint handles, pass through real-world identifiers.** Provider
  handles (offer, segment, passenger, seat, service refs) are never exposed
  raw; PNRs, ticket numbers, and airline/airport codes are returned verbatim.
- The **three kinds of id** — *issued* (`AGW` order ids), *derived*
  (`prefix + UUIDv5`, deterministic and recomputable), and *passed through*.
- The **five-stage lifecycle** from provider values in the requester, through
  minting in the response adapter, to inbound resolution by `AGWIDResolver`.
- The **layer rules**: minting and resolution live only in `adapters/`, `store/`
  holds provider values, and every dispatch path must load its mapping source.
- **Namespaces** (one per entity type) and the **seeding** rules for entities
  with no single provider reference, such as seats and services.
- **Known non-conformance** — deliberate, tracked exceptions that are to be
  closed and are never precedent.

## Rules of engagement

- All platform layers MUST conform to the exact names, transitions, and id
  shapes defined here. No inventing states, renaming identifiers, or adding
  transitions outside a PR that updates the relevant spec.
- Behaviour changes and spec changes land in the **same PR** — the spec is
  updated first, never after the fact.
- If any code, doc, or skill contradicts these files, this repo wins. The order
  lifecycle spec supersedes the retired skill `agw-v2-order-status-lifecycle`.
- The order and proposal machines are **independent**. Read them together for
  display; never let one drive the other.
