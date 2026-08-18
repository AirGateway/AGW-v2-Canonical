# Identifiers — the AGW ID policy

> **Normative for all new code.** Every identifier the API returns is either
> platform-issued, derived from a provider reference, or a real-world identifier
> passed through deliberately. Provider-internal handles are **never** exposed raw.
> If code contradicts this file, the code is wrong — except where *Known
> non-conformance* records a deliberate, tracked exception.

## The rule

**Mint handles. Pass through real-world identifiers.**

A **handle** is an identifier whose only purpose is to be echoed back to us: an
offer id, an offer item id, a segment or bound reference, a passenger reference, a
seat or service id, a list key, a shopping response id. It means nothing outside the
provider's own session. Exposing one raw leaks provider internals, couples every
client to a per-provider format, and — worse — makes the client the carrier of state
we then act on.

A **real-world identifier** has meaning outside our platform and the agent or
traveller needs it verbatim: the airline record locator (PNR), ticket and EMD
numbers, airline and airport codes, currency codes, fare basis codes, RFIC/subcodes.
These are data, not handles. Never mint them.

## Three kinds of id

| Kind | How it is produced | Examples |
|---|---|---|
| **Issued** | We generate it; it is the row's primary identity and is stable forever | order `AgwId` (`AGW` prefix, generated hub-side by `CreateOrder`) |
| **Derived** | `prefix + UUIDv5(namespaceFor(prefix), seed)` — deterministic, recomputable, needs no storage of its own | `offer-`, `bound-`, `seg-`, `pax-`, `seat-`, `service-` (`adapters/agwid.go`) |
| **Passed through** | Returned verbatim | PNR / booking reference, ticket numbers, coupons, airline & airport codes |

When a provider handle has genuine diagnostic value to the client, expose it under an
explicitly external name **alongside** the AGW id — never as the primary id. The
precedent is already in the order response: `orderID` is the AGW id and
`externalOrderID` is the provider's own order id
(`adapters/huborderretrieve.go:33-35`). Follow that shape.

## The lifecycle

Five stages, one direction. Any new entity that needs an id follows this exactly.

1. **Requester returns provider values** — discrete typed fields, never an encoded
   blob. The requester never learns that AGW ids exist.
2. **API persists the provider values** — in the store, keyed so they can be found
   again. The store is the mapping table.
3. **API mints the AGW id in the response adapter** — the *only* place a
   client-facing id is produced. Use the `idMappingsCollector`
   (`adapters/apioffer.go:100`) so the same provider ref yields the same AGW id
   across every response that mentions it.
4. **API resolves inbound** — `AGWIDResolver` (`adapters/agwidresolver.go`) turns
   AGW ids back into provider refs before validation and dispatch. Unknown ids pass
   through unchanged, which is what keeps a prefix rollout backward compatible.
5. **Requester receives provider values** — exactly what it returned in stage 1.

Two mapping sources, both already in use — prefer the second:

- **Stored pairs**: `domain.Order.IdMappings` holds AGW↔provider pairs for bounds,
  segments and passengers (`adapters/orderidmappings.go`). Survives a namespace or
  prefix change, at the cost of a payload to keep in sync.
- **Recompute**: hash the persisted provider ref on read to rebuild the map
  (`AGWIDResolver.AddSegmentRef`, `agwidresolver.go:68`). No storage, nothing to
  keep in sync, one deterministic function. Correct default. Only reach for stored
  pairs when the entity outlives a plausible namespace change — orders live for
  months, a shopping session does not.

### Layer rules

- Minting happens **only** in `adapters/`, in a response converter.
- Resolution happens **only** in `adapters/`, called from `service/` before
  validation and dispatch.
- `store/` holds provider values. Never persist a minted id as the authority for
  anything; it is always recomputable.
- Because a minted id is meaningless without its mapping source, **every dispatch
  path must load it.** Resolve inside the request converter that already takes the
  stored data (`RequestSeatsToRv2`, `RequestServicesToRv2`) rather than leaving it a
  separate step a future call site can forget. This is the failure mode to design
  out, not document.

### Namespaces

One namespace per **entity type**, seeded from a fixed string under
`uuid.NameSpaceURL` (`adapters/agwid.go:15-19`), so two entity types that hash the
same provider string still produce distinct ids. Never share a namespace across two
id spaces that are not interchangeable: the ids become visually identical and
mutually unusable, and a client that mixes them gets a misleading "not available"
error instead of an obvious mismatch.

`agwID` strips the UUID's dashes, so the body remains valid input to Postgres' `uuid`
type once the prefix is removed. Keep that. `StripAGWPrefix` must list every prefix.

### Seeding

Most ids hash the provider's own reference. Seats and services have no single
provider reference, so they hash the values that **determine the provider request**
— `offerItemID` + row + column for a seat, `offerItemID` + segments for a service
(`AGWSeatID` / `AGWServiceID`). Two consequences worth preserving:

- Any two entries that hash alike would also dispatch identically, so a seed
  collision cannot mis-book.
- The passenger and the offer/response ids are deliberately **excluded**: one offer
  item covers every eligible passenger (the request names the chosen one
  separately), and excluding the offer ids means re-querying a seat map does not
  churn ids the client already holds — resolution goes through the persisted map,
  which always carries the current offer coordinates.

A provider that exposes no offer item of its own (Kyte) seeds on its own per-entry
reference instead; without that fallback every such entry would seed alike.

## Known non-conformance

Tracked, deliberate, and to be closed — not precedent.

- **Order-view `seatID` / `serviceID`** are a different id space from the catalogue
  ones: `ServiceResponse.ServiceID` is `domain.Service.ServiceId`
  (`adapters/huborderretrieve.go:559`) and `SeatResponse.SeatID` is
  `domain.Seat.ListKey` (`:499`). Providers fill these inconsistently — a
  service-definition ref (BA, Iberia, Amadeus, Navitaire, FLX, LA, BT), an order item
  id (AFKLM), or a synthetic counter (`SRV%d`, AER). Nothing consumes them today
  because there are no remove endpoints. When `AirOrderRemoveServices` /
  `AirOrderRemoveSeats` land they will need to identify a service *already on the
  order*; decide that id space then, and give it its own namespace — never the
  catalogue's.
- **`AirOrderCreate` and `AirOrderIssue` accept `seats` / `services` and drop them.**
  Both request schemas carry the fields and both resolve the ids correctly, but
  neither populates `OfferRef.Ancillaries`, so the selection never reaches the
  airline. Ancillaries-at-create are delivered only by the extra-OfferPrice path in
  `ordercreateandissue` (`UX`, and services-only for `AY`/`SQ`/`LO`/`EY`); every
  other provider drops them there too. Independent of this policy and deliberately
  left as-is.
