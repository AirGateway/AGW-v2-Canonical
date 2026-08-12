# AGW Platform — Canonical Order State Machine

This repository holds the **single source of truth** for order **states**,
**workflows**, and **events** on the AirGateway platform.

The entire specification lives in [CLAUDE.md](CLAUDE.md). That filename is
intentional: it doubles as the project instructions loaded by
[Claude Code](https://claude.com/claude-code), so any AI-assisted work in this
repo (or any repo that vendors this file) is automatically bound by the
canonical contract.

## What the spec defines

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

## Rules of engagement

- All platform layers MUST conform to the exact names and transitions in
  [CLAUDE.md](CLAUDE.md). No inventing states, renaming identifiers, or adding
  transitions outside a PR that updates that file.
- Behaviour changes and spec changes land in the **same PR** — the spec is
  updated first, never after the fact.
- This file supersedes the retired skill `agw-v2-order-status-lifecycle`; if
  any code, doc, or skill contradicts it, this repo wins.
