# AGW Platform — Canonical Specs

This repository holds the **single source of truth** for two platform-wide
contracts: order **states, workflows, and events**, and the **identifier
policy** governing every id the API returns.

Each spec is normative on its own terms. Together they define what the platform
calls things and what it is allowed to do to them.

## [ORDERSTATEMACHINE.md](ORDERSTATEMACHINE.md) — order lifecycle

- The **12 canonical order statuses** (`pending`, `issued`, terminal EXIT
  statuses, transitional `part_flown`, and the recoverable `blocked`/`unknown`).
- The **Workflows → Transitions → Events** contract table — the exhaustive,
  normative list of valid `(from, to)` pairs and the event each one emits.
- The **coupon model** used to derive the flown EXIT statuses
  (`all_flown`, `some_flown`, `all_no_show`).
- The **`AirOrderChangeNotif`** inbound callback and its canonical `TYPE` enum
  (informational only — never a status change).
- The **API Requester** layer: how provider-specific request sequences collapse
  atomically into exactly one canonical transition and one event.
- The **layer contract** binding persistence, controller/service, API, front
  end, and tests to the exact canonical names.

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
