# AGW Platform — Canonical Specs

This repository holds the **single source of truth** for six platform-wide
contracts: order **states, workflows, and events**, proposal **statuses and
transitions**, the **traveller and company profiles** an agency curates, the
**identifier policy** governing every id the API returns, the **namespace
and naming policy** governing what the API calls things in the first place, and
the **presentation policy** governing how a front end puts an item in front of an
agent.

Each spec is normative on its own terms. Together they define what the platform
calls things, what it is allowed to do to them, and how an agent sees them.

## [ORDER-STATE-MACHINE.md](ORDER-STATE-MACHINE.md) — order lifecycle

- The **12 canonical order statuses** (`Pending`, `Issued`, terminal EXIT
  statuses, transitional `PartFlown`, and the recoverable `Blocked`/`Unknown`).
- The **Workflows → Transitions → Events** contract table — the exhaustive,
  normative list of valid `(from, to)` pairs and the event each one emits.
- **One diagram per workflow**, not a single aggregate: each drawing carries the
  starting status, the request sequence that fulfils the workflow, the resulting
  status, and the one event emitted — plus separate drawings for the detection
  transitions and the inbound callback.
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
- **The trail is a thread and a feed.** `TravellerMessaged` / `AgencyMessaged` are
  the two halves of the conversation — there is no separate messages resource — and
  **activity** is the same entries aggregated across every proposal, newest first,
  flagged seen or unread. Seen is the **agency's, not each agent's**, held as a
  watermark on the entry **id** and never on a timestamp, and moved only by an
  explicit acknowledge — never by reading.

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

## [NAMING.md](NAMING.md) — namespaces and operation naming

- The rule: **a namespace names the domain of the *thing*, not the domain of its
  contents.** `/v2/air` is for what varies by provider, offer or order; everything
  else gets its own root namespace (`/v2/proposals`, `/v2/profiles`, `/v2/agency`).
- The corollary that is easiest to get wrong: **a container is named for the
  container, not for what is put inside it.** A proposal carries air options today
  and may carry a hotel tomorrow, so it is `proposal`, never `airProposal`.
- The test for anything new: **"does an airline have to exist for this thing to
  exist?"** If not, it does not go under `air`.
- The **operation naming shape** — `operationId` is `camelCase`
  `<namespace><Resource><Verb>`, `summary` is the same string in `PascalCase`, the
  resource segment is dropped when the namespace *is* the resource, and a
  sub-resource keeps the **verb last** (`proposalOptionOffer`, not
  `proposalOfferOption`).
- The **schema rule**: a component carries its operation's prefix, and carries `Air`
  only when the payload is itself air content.
- The **layer contract** binding the spec, the generated Go, the hub API, the front
  ends and these specs to one canonical name.
- **How to rename an operation that has already shipped** — outright when it is not
  in production, `deprecated: true` with the reason and shared schema `$ref`s when it
  is.
- **Known non-conformance** — deliberate, tracked exceptions, never precedent.

## [PRESENTATION.md](PRESENTATION.md) — how a front end shows an item

- The rule: **an item is presented as a pair** — the identifier it is known by, plus
  one navigation affordance. A **link** where the item has a page of its own, a
  **hover** where it has an identity worth revealing but nowhere to go. Never one
  without the other.
- The **table**: an **order** shows its AGW order id and links to `/orders/:orderId`
  in BookingPad; a **user** shows its AGW user id and reveals the person on hover; a
  **proposal** shows its proposal id and links to `/proposals/:proposalId`. The table
  is deliberately open — more items are expected.
- The practice rules that follow: the id **is** the visible text and stays copyable,
  a link is a **real anchor** (⌘-click and "copy link address" must work), a link is
  **unconditional**, a hover must also reach the accessible name because touch and
  keyboard have no hover, and the pair does not change between list, detail and
  dialog.
- **Adding an item**: one row, the affordance picked from what exists today, spec and
  front end in the same PR.
- The **layer contract** binding BookingPad web, the Expo app and the traveller app
  to the same pairs — and forbidding a front end from prettifying an id to show it.
- **Known non-conformance** — today a user's hover shows an **email**, because no
  agent has a name anywhere on the platform.

## Rules of engagement

- All platform layers MUST conform to the exact names, namespaces, transitions,
  and id shapes defined here. No inventing states, renaming identifiers, adding
  transitions, or introducing a path or `operationId` outside a PR that updates
  the relevant spec.
- Behaviour changes and spec changes land in the **same PR** — the spec is
  updated first, never after the fact.
- If any code, doc, or skill contradicts these files, this repo wins. The order
  lifecycle spec supersedes the retired skill `agw-v2-order-status-lifecycle`.
- The order and proposal machines are **independent**. Read them together for
  display; never let one drive the other.
- A screen that shows an item shows it the way [PRESENTATION.md](PRESENTATION.md)
  says, in every surface that mentions it.
