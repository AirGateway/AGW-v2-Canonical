# Namespaces and Operation Naming — Single Source of Truth

> **Canonical spec — single source of truth.** This document is *the* authoritative
> definition of how AGW API V2 **namespaces** its surfaces and **names** its operations
> and schemas. All layers — AGW API V2, hub persistence and its admin API, BookingPad,
> the traveller app — MUST conform. Do not introduce a namespace, an `operationId`, or a
> component schema that contradicts it without updating this file via PR.
>
> If any code, doc, or skill contradicts this file, this file wins and the other must be
> corrected.

## The rule

**A namespace names the domain of the *thing*, not the domain of its contents.**

`/v2/air/**` is for **air content**: anything whose meaning, shape or availability varies
by provider, offer, or order. Shopping, offers, orders, and every servicing action on an
order live there, and they belong there because an airline is party to each one.

Anything that does **not** vary by provider, offer or order gets **its own root
namespace**. Today that is:

| Namespace | Holds | Why it is not `air` |
|---|---|---|
| `/v2/air` | shopping, offers, orders, order servicing | an airline is party to every one |
| `/v2/proposals` | proposals and the options attached to them | a container an agent fills; no airline is party to raising one |
| `/v2/profiles` | travellers, companies | a roster the agency curates; no airline is involved in creating one |
| `/v2/agency` | the agency's own agents, presets, remark templates | agency configuration; identical whoever the provider is |

### A container is named for the container, not for what is put inside it

This is the corollary that is easiest to get wrong, and the one that produced the
`AirProposal*` mistake this document exists to prevent.

A **proposal is a container**. An agent fills it with options. Today those options are
air orders, because air is what the platform sells today — but the proposal itself has no
provider, no offer and no PNR. It has a traveller, a request, a status, an assignee and a
history, and every one of those would read identically if the option attached were a
hotel, a rail leg or an insurance policy.

**Naming a container after its current contents bakes today's product scope into a public
URL.** The day a proposal carries a hotel option, `POST /v2/air/proposals/{id}/accept` is
not merely inelegant — it is *wrong*, and it is wrong in the one place the platform cannot
cheaply correct: a path partners have integrated against.

So: the container is `proposal`, never `airProposal`. The same test applies to anything
added later. Ask **"does an airline have to exist for this thing to exist?"** If the answer
is no, it does not go under `air`.

### This has been decided twice before

The rule is not new; it is being written down. The platform already moved two surfaces out
of `air` for exactly this reason, and the spec still carries the reasoning:

- `GET /v2/air/agency/agents` is `deprecated: true`, replaced by `GET /v2/agency/agents`,
  because *"an agency's roster is not air content: it does not vary by provider, offer or
  order, so it moved out of the Air namespace."*
- [PROFILES.md](PROFILES.md) records the same call: *"profiles live under `/v2/profiles`,
  outside `air`, because no airline is involved in creating one."*

Proposals are the outlier that never got the treatment. They are being brought into line.

## Operation naming

- **`operationId`** — `camelCase`, shaped `<namespace><Resource><Verb>`.
- **`summary`** — the identical string in `PascalCase`. Case is the only difference, and
  it is normative.
- **The namespace segment is the root path segment, singular.** `/v2/profiles/travellers`
  → `profileTravellerList`. `/v2/agency/agents` → `agencyAgentList`. `/v2/proposals/{id}`
  → `proposalRetrieve`.
- **The resource segment is omitted when the namespace *is* the resource.**
  `/v2/proposals/{id}/accept` → `proposalAccept`, not `proposalProposalAccept`.
- **A sub-resource takes its own segment, and the verb stays last.**
  `POST /v2/proposals/{id}/options` → `proposalOptionOffer`, **not**
  `proposalOfferOption`. The verb-last order is what keeps every operation on a resource
  sorting together in the docs, and it is why `profileTravellerCreate` reads the way it
  does.
- **Verbs are the CRUD set where the operation is CRUD** — `Create`, `List`, `Retrieve`,
  `Update`, `Delete` — and the domain's own word where it is not: `Accept`, `Decline`,
  `Assign`, `Confirm`, `Withdraw`, `Discard`.

## Schema naming

- A component schema carries **the same prefix as the operation it serves**:
  `proposalAccept` → `ProposalAcceptRequest` / `ProposalAcceptResponse`.
- A schema carries the `Air` prefix **only when the payload is itself air content** —
  an offer, an order, a segment, a fare. `AirProposalResponse` was wrong because the
  proposal envelope is not air content, even though an option inside it points at an order
  that is.
- Shared domain types are named for the domain object alone: `ProposalOption`,
  `ProposalStatus`, `Traveller`.

## Layer contract

The canonical name is set by the AGW API V2 `operationId`, and every layer mirrors it:

| Layer | Follows |
|---|---|
| **AGW API V2 spec** | `operationId` / `summary` / component schemas as above. **Source of truth.** |
| **AGW API V2 Go** | ogen-generated from the spec — never hand-named. Handler methods take the generated name. |
| **Hub API** | the resource word only, never the namespace prefix: hub groups are `/agw/proposals`, `/admin/proposals`, and hub types are `Proposal*`. Hub is *behind* the namespace, so it does not repeat it. |
| **BookingPad / traveller app** | the path and the schema names verbatim; no local aliasing that reintroduces a dropped prefix. |
| **Canonical specs** | transitions are named by the `operationId` that causes them ([PROPOSALS.md](PROPOSALS.md), [ORDER-STATE-MACHINE.md](ORDER-STATE-MACHINE.md)). |

## Renaming an operation that has already shipped

Renaming a public operation is a breaking change and is governed by where it has shipped:

- **Not in production** (`main`/`sandbox` only) — rename outright. No alias. An alias kept
  for callers who do not exist is permanent dead surface, and it doubles the generated
  client for nobody.
- **In production** — keep the old path with `deprecated: true`, a description naming the
  replacement **and the reason**, and the *same* response schema `$ref`s as its
  replacement, so the two can never drift. `AirAgencyAgents` is the worked example.

Either way the rename lands as **one PR per repository**, spec first, and the client
deploys follow the API deploy in the same window.

## Known non-conformance

Deliberate, tracked exceptions. These are to be closed, and are **never** precedent for
new surfaces.

| Surface | Non-conformance | Status |
|---|---|---|
| `GET /v2/air/agency/agents` (`airAgencyAgents`) | agency roster under `air` | `deprecated: true`, replaced by `agencyAgentList`; to be removed |
| `/v2/air/proposals*` (`airProposal*`) | container named after its contents | being renamed to `/v2/proposals*` / `proposal*`; nothing in production, so no alias is kept |
