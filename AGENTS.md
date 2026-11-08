# AGW-v2-Canonical — memory & compliance checkbook

This repo is the **single source of truth**. If any code, doc, or skill contradicts
these files, these files win. This memory distils the six specs into mechanically
checkable rules and records how to audit the platform repos against them.

Specs: `ORDER-STATE-MACHINE.md` · `PROPOSALS.md` · `PROFILES.md` · `IDS.md` ·
`NAMING.md` · `PRESENTATION.md`. Read the spec, not this file, before changing
behaviour. Behaviour + spec changes land in the **same PR, spec first**.

## Repos under audit

| Repo | Role | Primary sources to check |
|---|---|---|
| `~/Workspaces/AirGateway/agw-api-v2` | AGW API V2 (Go) | `specs/ndcjsonapiv2.openapi.yaml`, `internal/api/{adapters,service,store,hubapi}/`, `internal/shared/{orderstate,proposal,constants}/` |
| `~/Workspaces/AirGateway/hub-api-v2` | hub persistence + `/agw` API | `domain/proposal.go`, `domain/traveler.go`, `domain/company.go`, `domain/order.go`, migrations, `testdata/database/seeds/` |
| `~/Workspaces/AirGateway/bookingpad-app-v2` | BookingPad web (Angular) | `src/app/features/{orders,proposals,profiles}`, `src/app/shared/utils/` |
| `~/Workspaces/AirGateway/agw-bp-app-v2` | BookingPad mobile (Expo) | status mirrors, links out via configured base URL |
| `~/Workspaces/AirGateway/bp-traveler-app` | traveller app | status mirror, traveller-visible trail only |

## Checkbook — the rules that are mechanically checkable

### 1. Order state machine (ORDER-STATE-MACHINE.md)

- [ ] Exactly **12 stored order statuses**, PascalCase: `Pending`, `Issued`,
      `Cancelled`, `Expired`, `Voided`, `Refunded`, `AllFlown`, `SomeFlown`,
      `AllNoShow`, `PartFlown`, `Blocked`, `Unknown`. Any other stored value is a bug.
- [ ] Only transitions listed in the Workflows → Transitions → Events table are
      accepted; unlisted `(from, to)` pairs are rejected, not coerced.
- [ ] `Blocked`/`Unknown` entry persists the **prior state** and recovery restores it.
- [ ] Case is normative: workflows `PascalCase` (`AirOrderVoid`), requests
      `camelCase` (`airOrderVoid`); never normalised into one another.
      `airOrderRetrieve` and `airOrderChangeNotif` are requests only — no PascalCase form.
- [ ] Events are PascalCase past tense (`AirOrderCreated`, `AirOrderIssued`,
      `AirOrderReissued`; `AirOrderSplit` conforms).
- [ ] `airOrderChangeNotif` never changes status; it emits `AirOrderChangeNotified`
      with a `TYPE` from the 35-value enum; fallback `UnknownNotification`.
- [ ] Flown states derive only from coupon aggregation (`flown`/`no_show`,
      lowercase snake_case at coupon level); `PartFlown` is never terminal.
- [ ] A workflow = exactly one transition + one event, commit-on-success only;
      partial provider sequences leave state unchanged.

### 2. Proposals (PROPOSALS.md)

- [ ] Exactly **7 proposal statuses**: `New`, `Open`, `Sent`, `Pending`, `Expired`,
      `Cancelled`, `Confirmed` (terminal: the last three).
- [ ] Exactly **6 option statuses**: `Offered`, `Approved`, `Rejected`,
      `RejectionFailed`, `Withdrawn`, `Expired`. Never written to `proposal.status`.
- [ ] Transitions match the normative table; anything else is `409`.
      Notably absent: `Sent`→`Open`/`New`, `Pending`→`Expired`, anything out of terminal.
- [ ] Independence: no order status change driven by a proposal; the **one**
      exception is an `Approved` option's order reaching `Issued` → proposal
      `Pending`→`Confirmed`, same transaction, `ProposalConfirmed` against `System`
      with `details.trigger: "OrderIssued"`.
- [ ] Guards: send needs ≥1 `Offered`; resend blocked while anything `Approved`;
      confirm needs ≥1 `Approved`; agency-side approve/cancel require a comment;
      cancel allowed from any live status by either side; cancel moves **all**
      non-terminal options (incl. `Approved`) to `Expired`; ≤3 offered options;
      attach requires order `Pending`, same agency, not already backing an option.
- [ ] Foreign traveller/agency scoping → **404, never 403**.
- [ ] History is append-only, PascalCase past-tense actions (`ProposalRaised`,
      `AgentAssigned`, `OptionOffered`, `OptionWithdrawn`, `ProposalSent`,
      `ChangesRequested`, `OptionApproved`, `ProposalCancelled`,
      `ProposalConfirmed`, `ProposalExpired`, `ExpiryChanged`, `NoteAdded`,
      `TravellerMessaged`, `AgencyMessaged`, `OptionCancelRequested`,
      `OptionCancelFailed`). No `updated_at` on entries; actor name stored as-of.
- [ ] Two trails: traveller sees everything except `NoteAdded`, `AgentAssigned`,
      `OptionCancelRequested`, `OptionCancelFailed`, and `Admin` actions.
- [ ] Activity feed: `proposalActivityList` / `proposalActivityAcknowledge` at
      `GET /v2/proposals/activity` / `POST /v2/proposals/activity/acknowledge`;
      no status filter; watermark is an **entry id, never a timestamp**, per
      (proposal, audience); forward-only; reads never move it; page size 50 max & default.
- [ ] Vocabulary declared once in hub `domain/proposal.go`; guards use constants,
      never string literals; taxonomy tables FK-enforced and reconciled at start-up;
      test seeds mirror every declared value.

### 3. Profiles (PROFILES.md)

- [ ] **3 traveller statuses**: `Provisional`, `Active`, `Inactive`;
      **2 company statuses**: `Active`, `Inactive`. None terminal. Nothing returns
      to `Provisional`; resolution only ever **mints** `Provisional` (or reactivates
      `Inactive`→`Active` when the invariant holds); nothing is created `Inactive` (422).
- [ ] `Active` invariant: email + name + surname, else `422`.
- [ ] Identity: email unique per **company** (case-insensitive, `citext` partial
      index); cross-agency ambiguity is `409`, never a guess.
- [ ] Tenancy: `travelers` has **no `agency_id` column** and must not gain one;
      scope derived `traveler → company → agency`, applied in SQL; foreign profile
      is `404`, never `403`.
- [ ] Spelling: **API `traveller` (British), DB `traveler` (US)** — never "fixed".
      API fields camelCase; hub `/agw` bodies snake_case; translation is agw-api-v2's job.
- [ ] No `nationality` field — it is `documents[].citizenshipCountryCode`.
- [ ] FrequentFlyer is `airlineCode` + `number`, not an alliance (known gap: `alliance` still stored).
- [ ] Validation responses: missing companyId 422, dup email 409, dup domain 409,
      delete-with-references 409 (RESTRICT), transition violations 409, unknown status 422.
- [ ] Remark templates: sub-resource of the company
      (`/v2/profiles/companies/{id}/remarks`), unpaged, `neededOnCreation` unique
      (second = 409 `AGW_profile_remark_mandatory_taken`, never `AGW_profile_incomplete`);
      legacy `companies.remarks` array never read/written by new code.
- [ ] `search` param is OR-across-fields; a single search box must NOT send
      `name`+`surname`+`email` together. `status` filter is comma-separated one key.
- [ ] Error mapping: 404 without RFC7807 body → 500 ("route not implemented");
      409 classified from hub's `detail` (`AGW_profile_email_taken` /
      `AGW_profile_domain_taken` / `AGW_profile_in_use`); 422 on writes →
      `AGW_profile_incomplete`, 422 on reads → 500; hub 401 never passed through.
- [ ] Booking is a snapshot, profile is the source: re-persist re-links but only
      **adopts** existing profiles (never creates on a read); no unlink.
- [ ] Derived read-only fields: `travellerCount`, `proposalCount`, `totalByStatus`,
      `firstOrderId` — computed, never columns.

### 4. IDs (IDS.md)

- [ ] Provider handles are **never** exposed raw. Real-world ids (PNR, ticket
      numbers, airline/airport/currency codes, fare basis) pass through verbatim —
      never minted.
- [ ] Derived ids: `prefix + UUIDv5`, dashes stripped, in `adapters/agwid.go`;
      one namespace per entity type; `StripAGWPrefix` lists every prefix.
- [ ] Minting only in `adapters/` response converters (via `idMappingsCollector`);
      resolution only via `AGWIDResolver` in `adapters/`, called from `service/`
      before validation/dispatch; `store/` holds **provider values only**; every
      dispatch path loads its mapping source inside the request converter.
- [ ] Provider handles with diagnostic value appear as `externalOrderID`-style
      fields **alongside** the AGW id, never as the primary id.
- [ ] Order id `AGW` prefix, issued hub-side by `CreateOrder`.
- [ ] Known non-conformance (do not copy, do not "fix" silently): order-view
      `seatID`/`serviceID` id space; `AirOrderCreate`/`AirOrderIssue` dropping
      `seats`/`services`.

### 5. Naming & namespaces (NAMING.md)

- [ ] Namespaces: `/v2/air` (airline is party), `/v2/proposals`, `/v2/profiles`,
      `/v2/agency`. Test: "does an airline have to exist for this thing to exist?"
- [ ] **No `airProposal*` / `/v2/air/proposals*`** anywhere new; container named
      for the container. `airAgencyAgents` stays only as `deprecated: true`.
- [ ] `operationId` camelCase `<namespace><Resource><Verb>`, **verb last**
      (`proposalOptionOffer`, not `proposalOfferOption`); resource segment dropped
      when the namespace is the resource (`proposalRetrieve`); `summary` =
      identical string PascalCase.
- [ ] Verbs: CRUD set (`Create`, `List`, `Retrieve`, `Update`, `Delete`) or the
      domain word (`Accept`, `Decline`, `Assign`, `Confirm`, `Withdraw`, `Discard`,
      `Acknowledge`).
- [ ] Schemas carry the operation prefix (`proposalAccept` →
      `ProposalAcceptRequest/Response`); `Air` prefix only for air payloads;
      shared domain types bare (`ProposalOption`, `ProposalStatus`, `Traveller`).
- [ ] Go code is ogen-generated from the spec — never hand-named.
- [ ] Hub uses the resource word only, no namespace prefix: `/agw/proposals`,
      types `Proposal*`.
- [ ] Rename policy: not in production → rename outright, no alias; in production →
      `deprecated: true` + reason + same `$ref`s. One PR per repo, spec first.

### 6. Presentation (PRESENTATION.md)

- [ ] Pair rule: id + one affordance. Order → link `/orders/:orderId`; proposal →
      link `/proposals/:proposalId`; user → hover (email, per known gap); company →
      name + link `/profiles/company/:companyId`; traveller → name + link
      `/profiles/traveller?traveller=:travellerId`.
- [ ] Links are real anchors (`routerLink`/`href`), unconditional, ⌘-clickable;
      hovers also reach the accessible name; pairs identical across surfaces.
- [ ] Cross-app links go through configured base URL — **never a hardcoded host**.
- [ ] Missing id renders as absence (em dash), never `undefined`, never a dead anchor.
- [ ] Front ends never prettify/derive an id; API returns the ids as named.

## How to run a compliance check

1. Pick the rule groups touched by the change (or run all for an audit).
2. Per repo, grep the primary sources above for: status string literals outside the
   closed sets; `airProposal` / `/v2/air/proposals`; camelCase/PascalCase drift on
   workflow vs request names; hardcoded hosts in front ends; `agency_id` on
   `travelers`; minting/resolution outside `adapters/`; string-literal statuses in
   hub guards.
3. Compare findings against **Known gaps / Known non-conformance** sections of each
   spec — a recorded gap is expected, not a new violation; anything else is a finding.
4. A finding is fixed **spec-first**: if the code is right and the spec is stale,
   the spec PR lands before/with the code PR.

## Recorded gaps baseline (re-verified by audit 2026-11-08 — specs are authoritative)

Closed since the specs were last updated (the specs' Known-gaps tables are stale on
these — fix spec-first): `airProposal*` rename has **landed** in agw-api-v2 (all 14
ops under `/v2/proposals*`, no `AirProposal*` schemas; `airAgencyAgents` is gone,
replaced by `agencyAgentList`); the **activity feed is built** in hub
(`bookings.proposal_history_watermarks`, `/agw/proposals/activity{,/acknowledge}`,
entry-id watermark, forward-only), in the agw-api-v2 spec, and in BookingPad (bell +
feed on the proposals inbox); **BookingPad agent-side approve/cancel exist**
(comment mandatory); **`companiesMockInterceptor` removed** (all mocks inactive in
every build).

Still open (expected, NOT new findings): no `proposalSend` / `proposalRequestChanges`
anywhere (first offer sets `Sent` implicitly); approve takes one option, not a list;
expiry sweep unwired (`ListExpired` has no caller, `ProposalExpired` never written);
expired options' airline holds not released; `frequentFlyerNumbers[].alliance`;
`travellerCode` uniqueness index; unpinned `gender`/`title`/`documentType` on the Air
surface; user hover shows email (no name in hub); company/traveller shown by name not
id; order-view `seatID`/`serviceID`; create/issue drop seats/services;
`bp-traveler-app` has no BookingPad links and `agw-bp-app-v2` is a scaffold — the
mobile layer of PRESENTATION.md is unbuilt.

Audit findings needing a spec-first decision (2026-11-08): hub declares an extra
history action **`OptionRejected`** (domain/proposal.go) beyond the canonical 16,
while canonical **`ChangesRequested`** is undeclared; agw-api-v2 spec
`airOrderRemarksRetrieve`/`airOrderRemarks` break the summary= PascalCase(operationId)
and verb-last rules; `/v2/air/agency/remark_templates` (`airAgencyRemarkTemplates`)
and `/v2/air/agency/presets` (`airAgencyPresets`) remain under `air` though they are
agency configuration; BookingPad ships `/profiles/traveller/:travellerId` (a page)
while PRESENTATION.md still says `?traveller=` query on the roster; BookingPad's
orders dashboard shows no orderID anchor (row `(click)` + bare PNR) citing a
"tracked exception" PRESENTATION.md does not record; hub enforces **at most one**
remark template per company while PROFILES.md specifies a list with a unique
`neededOnCreation` flag.
