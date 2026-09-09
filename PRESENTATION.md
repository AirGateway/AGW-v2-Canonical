# Presentation — how a front end shows a platform item

> **Normative for every AGW front end.** Wherever a screen puts a platform item in
> front of an agent — a list row, a detail header, a dialog, a badge, a history
> entry, a toast — it shows the pair defined here and nothing else. This file does
> not describe an API contract; it describes what the agent sees, so that the same
> item looks and behaves the same in every surface that mentions it. If code
> contradicts this file, the code is wrong — except where *Known non-conformance*
> records a deliberate, tracked exception.

## The rule

**An item is presented as a pair: the identifier it is known by, and one navigation
affordance attached to it.** A **link** where the item has a page of its own; a
**hover** where it has an identity worth revealing but nowhere to go.

Never one without the other, and never a third thing in place of either. The
identifier is what the agent quotes to an airline, pastes into a ticket, and searches
for; the affordance is what saves them from having to. Showing a bare id makes every
cross-reference a manual copy-paste hunt. Showing a friendly label instead of the id
makes the thing unquotable. The pair is the unit.

## The table

| Item | Field shown | Affordance | Target / content |
|---|---|---|---|
| **Order** | the AGW order id (`orderID`) | **link** | the order in BookingPad — `/orders/:orderId` |
| **User** (agency agent) | the AGW user id (`id` from `agencyAgentList`) | **hover** | the user's full name — see *Known non-conformance* |
| **Proposal** | the proposal id (`id`) | **link** | the proposal in Proposals — `/proposals/:proposalId` |
| **Company** (profile) | the company **name** — see *Known non-conformance* | **link** | the company's page in Profiles — `/profiles/company/:companyId` |
| **Traveller** (profile) | the traveller's **full name** — see *Known non-conformance* | **link** | the Profiles roster with that profile open — `/profiles/traveller?traveller=:travellerId` |

More items will be added here. One item is one row; see *Adding an item*.

## What the rule means in practice

- **The identifier is the visible text.** Not a friendlier label, not a sequence
  number, not "this order". If space forces a visual truncation, the full value must
  still be selectable and present in the accessible name — an id an agent cannot copy
  in full has failed at its only job.
- **A link is a real anchor.** `routerLink` / `href` on an `<a>`, never a click
  handler on a `<span>` or a `<div>`. Middle-click, ⌘-click, "open in new tab" and
  "copy link address" must all work, because triaging is a many-tabs activity.
- **A link is unconditional.** If the item exists and its id is present, the id
  links. A front end does not decide on the agent's behalf which of the items on
  screen are worth opening — an agent following a trail backwards needs the ones the
  screen considers uninteresting most of all.
- **A hover is progressive disclosure, never the only channel.** Touch has no hover
  and neither does a keyboard. Whatever the hover reveals must also reach the
  accessible name (`title` / `aria-label`) and appear on focus, not only on
  `:hover`.
- **The pair does not change between surfaces.** The same order id links the same way
  in the orders dashboard, on a proposal option, inside a dialog, and on a card in the
  proposals activity feed. A surface may add context around the pair; it may not swap
  the affordance or drop it. The feed is the case that tempts an exception — a whole card
  is clickable, so the id looks decorative — and it is not one: the card may carry a click
  as well, but the id inside it is still the anchor, or ⌘-clicking a request out of a feed
  to triage it in a second tab stops working.
- **Cross-app links go through configuration.** A front end that cannot route to the
  item itself — the Expo app, the traveller app — links to BookingPad through the
  environment's configured base URL. Never a hardcoded host: `sandbox`, staging and
  prod are different origins and a baked-in one is a bug that only shows up in
  production.
- **A missing id renders as absence, not as a broken affordance.** An em dash or
  nothing at all. Never `undefined`, never an anchor whose target is a missing id.

## Adding an item

The list is deliberately open — "more items to come" is the expected state, not a
gap. To add one:

1. Add **one row** to the table: the item, the exact field, the affordance, and the
   target route or hover content.
2. Pick the affordance from what exists, not from what would be nice. An item gets a
   **link** only if it has a page today. If it does not, it gets a **hover** and a
   row in *Known non-conformance* saying what a link is waiting on.
3. Land the row and the front-end change in the **same PR**, the way every other
   spec in this repo requires.

## Layer contract

| Layer | What it owes |
|---|---|
| **`bookingpad-app-v2`** (Angular, the BookingPad web GUI) | The canonical implementation. Routes are `/orders/:orderId` (`features/orders/orders.routes.ts`), `/proposals/:proposalId` (`features/proposals/proposals.routes.ts`), `/profiles/company/:companyId` and `/profiles/traveller?traveller=:travellerId` (`features/profiles/profiles.routes.ts` — a traveller has no page of its own; the roster opens its profile dialog); link with `routerLink`, never a manual string. |
| **`agw-bp-app-v2`** (Expo, the BookingPad mobile app) | The same pairs, with the same identifiers. Where the mobile app has no screen for the item, it links out to the BookingPad web app through its configured base URL. |
| **`bp-traveler-app`** | The same pairs for whatever it shows a traveller, minus anything a traveller has no business seeing. Agency-internal ids — the AGW user id above — are not traveller-facing. |
| **AGW API V2** | Returns the identifiers the table names, under the names it names them by. A front end never derives, formats or prettifies an id to display it; see [IDS.md](IDS.md). |

## Known non-conformance

Deliberate, tracked exceptions. Each is to be closed. **None is precedent.**

| Exception | Status |
|---|---|
| **Company and Traveller show a name, not their id.** Profile ids are unprefixed UUIDs ([IDS.md](IDS.md) — the `public.companies` / `public.travelers` primary keys, deliberately not minted handles), and nobody quotes one to an airline or pastes one into a ticket; a UUID as the visible text would make every roster mention unreadable and quotable of nothing. The name is shown and the id travels in the link target and the accessible name (`aria-label` names the id), so it stays copyable. Closing this is an IDS decision — prefixed profile handles — not a front-end one. | Open, deliberate |
| **A user's hover cannot show a full name, because no agent has one.** Hub's `agents` table holds an email and nothing else human-readable, so `agencyAgentList` returns `id`, `email`, `role` and `active` — and `email` is the whole of what any surface can reveal. Until hub gains a name field, the hover shows the **email**, which is the closest thing to an identity the platform has. Closing this is a hub schema change, not an API one. | Open |
