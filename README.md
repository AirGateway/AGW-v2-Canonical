# AGW Platform — Canonical Specs

This repository holds the **single source of truth** for three platform-wide
contracts: order **states, workflows, and events**, proposal **statuses and
transitions**, and the **identifier policy** governing every id the API
returns.

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
- The **migration table** from the 7-status set currently on `sandbox`, which is
  not this machine.

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
