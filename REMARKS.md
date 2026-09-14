# Remarks — Single Source of Truth

> **Canonical spec — single source of truth.** This document is *the* authoritative
> definition of the **remark template language**, the **two remark kinds** an order
> carries, and the **rendering contract** that turns a filled template into the text
> sent to a provider. All layers — hub persistence, AGW API V2, BookingPad, the
> traveller app — MUST conform to the exact grammar, token set and rendering rules
> defined here. Do not invent placeholder types, add autofill tokens, or change what
> a line renders to without updating this file via PR.
>
> If any code, doc, or skill contradicts this file, this file wins and the other must
> be corrected.

## What a remark is, and what it is not

A remark is **the agency's own text, carried on an order**. It exists for the agency's
mid/back-office — accounting, invoicing, MIR/AIR export — and for the PNR lines an
agency has always written by hand in a GDS.

Two things follow, and everything below depends on them:

- **A remark is authored by the agency, not by the platform.** The platform supplies a
  *template language* and fills in what it already knows. It does not decide what a
  remark says, and it MUST NOT reformat, wrap, trim or reorder the text an agency wrote.
  A template's line breaks are the PNR's line breaks.
- **A remark is not a provider field.** Nothing in a remark changes a booking, a price,
  a service or a status. A remark that fails to render is a rendering defect, never a
  booking failure.

A remark is distinct from **provider remarks** — free text an airline returns on order
retrieve. Those are provider output, are never templated, and are presented as warnings.
They share only the English word.

## The two remark kinds

An order carries **at most one remark of each kind**, and never more.

| Kind | Templates come from | Serialises into | Authored by |
|---|---|---|---|
| **Agency** | `GET /v2/air/agency/remark_templates` — agency configuration, one list per agency | `agencyRemarks` | AirGateway, as agency configuration |
| **Company** | `GET /v2/profiles/companies/{id}/remarks` — per company, read when the agent picks the company | `companyRemarks` | **The agency itself**, in the Profiles screen |

Both kinds use **the same language, the same rendering contract and the same wire
shape**. They differ only in where their templates come from and which body field they
land in. A layer that special-cases one kind's grammar is non-conformant.

**Company templates are agent-authored.** An agency creates, edits and deletes a
company's remark template from BookingPad without AirGateway involvement — at most
one per company, so there is nothing to position (see *Mandatory templates* below).
This is why the grammar below is normative and must stay small: the people writing
templates are travel agents, not integrators.

> **Naming.** The canonical word is **company**, matching
> [PROFILES.md](PROFILES.md). `corporate` is the retired BookingPad v1 term and MUST
> NOT appear in an API field, a spec, or a new front end.

## The template language

A template is a `\n`-separated string. It is read **line by line**; a line is one PNR
line. There is **no escape syntax**: `RM\*ACESAL` is literal text, and a brace cannot be
escaped out of being a placeholder.

### Line kinds

A line is exactly one of these, decided on the **trimmed** line:

| Kind | Recognised by | Shown to the agent | Rendered to output |
|---|---|---|---|
| **Block delimiter** | the whole trimmed line is `{{#passengers}}` or `{{/passengers}}` | no | no |
| **Comment** | starts with `#` | yes, as a note | **yes, verbatim** |
| **Blank** | empty or whitespace only | yes, as spacing | **no** |
| **Field line** | contains at least one placeholder | yes, as inputs | conditionally — see [Rendering](#the-rendering-contract) |
| **Static** | anything else | yes, as text | yes, verbatim |

Two rules here are load-bearing and easy to get wrong:

- **A block delimiter is only a delimiter on a line of its own.** `{{#passengers}}`
  appearing inline is not a block — it renders as literal text.
- **A comment is exported.** `#` marks a line the agent reads as a note; it does **not**
  hide the line from the provider. A layer that tells an agent otherwise is wrong — see
  [Known non-conformance](#known-non-conformance).

### Placeholder grammar

```
{ label : type ( args ) suffix }
```

- **`label`** — required. The variable name. MUST NOT contain `:`, `{` or `}`.
  Leading/trailing whitespace is trimmed.
- **`type`** — optional, letters and `-` only, **case-insensitive**. Absent means `str`.
- **`args`** — optional, inside `(…)`. Length bounds, decimal places, or list values.
- **`suffix`** — optional, `!` or `?`.

A placeholder that does not match this shape is literal text, not a field.

### Types

The type set is **closed**. These five are the whole language:

| Type | Args | Widget | Validation |
|---|---|---|---|
| `str` (or `string`, or omitted) | `(max)` or `(min,max)` | text | character length bounds |
| `int` | `(max)` or `(min,max)` | text, digits only | `^\d+$`, plus length bounds |
| `float` | `(decimals)` | text, digits and one `.` | at most *decimals* digits after the point |
| `float` | `(min,max)` | text, digits and one `.` | parses as a number, plus length bounds |
| `list` | `(a,b,c)` | single-select | value is one of the options |

- **`float` arity is the discriminator.** `float(2)` is *two decimal places*;
  `float(2,5)` is *length between 2 and 5*. One number means precision, two mean length.
  This is deliberate and MUST NOT be "harmonised" with `str`/`int`.
- **`str(N)` sets a maximum only.** There is no max-less minimum: write `str(N,M)`.
- **List options are trimmed and empties dropped.** An option MUST NOT contain `,` or `)`.
- **An unknown type renders as a plain text input.** The language never rejects a
  template — a typo becomes a field the agent types into. See
  [Known non-conformance](#known-non-conformance) for why this matters during the v1 → v2
  migration.

### Modifiers

| Suffix | Name | Agent | Output |
|---|---|---|---|
| `!` | **required** | MUST fill it before the remark can be saved | line always renders |
| `?` | **optional** | may leave it empty | empty ⇒ **the whole line is dropped** |
| *(none)* | **regular** | may leave it empty | empty ⇒ **the whole line is dropped** |

**`?` and no-suffix render identically.** The distinction is documentary: `?` tells the
agent the emptiness is expected. Only `!` changes behaviour, and it changes it by
gating the save. A spec or UI that promises a bare field renders with a gap is wrong.

### Autofill tokens

A placeholder whose type is one of these is filled by the front end and presented
**disabled** — the agent sees the value and cannot change it.

The token set is **closed. These four are all of them.**

| Token | Value | Scope |
|---|---|---|
| `origin` | IATA code of the **first bound's** departure airport | whole template |
| `destination` | IATA code of the **first bound's** arrival airport | whole template |
| `travelerReference` | the passenger's traveler reference | passenger block only |
| `number` | the passenger's 1-based position | passenger block only |

- Tokens are matched **case-insensitively** (`{o:Origin}` works).
- **First bound, not final destination.** On a WAW→BUD→WAW round trip,
  `{origin:origin}-{destination:destination}` renders `WAW/BUD`. This is the confirmed
  rule, not an accident: a remark names the outbound the agency sold.
- Outside a passenger block, `travelerReference` and `number` render empty.
- **Adding a token is a spec change.** A front end MUST NOT resolve a token this table
  does not list; anything else is a text input the agent fills.

### Passenger blocks

```
{{#passengers}}
RM*PAX/{fare:float(2)}/{ref:travelerReference}
{{/passengers}}
```

- Every line between the delimiters repeats **once per passenger**, in passenger order.
- A block MAY contain **several lines**. All of them repeat together, per passenger.
- Blocks **MUST NOT nest**; a nested block's lines are dropped.
- An **unclosed** block swallows every remaining line of the template and is still
  expanded. This is defensive, not a feature — a template MUST close its block.
- A template SHOULD contain at most one block. Several are expanded independently and
  each repeats the full passenger list.

Inside a block, a field's variable key is suffixed with the traveler reference:
`fare` becomes `fare-ADT0`, `fare-ADT1`, … Outside a block, the key is the bare label.

**Duplicate keys share one control.** The same label twice on one line — or in one
block iteration — is one input rendered once and substituted in both positions.

## The rendering contract

Rendering a filled template produces two things: the **output** string sent to the
provider, and the **variables** map that produced it. The rules are exhaustive:

1. **Blank lines are dropped.** A template cannot emit an empty line.
2. **Static and comment lines are kept verbatim.**
3. **A field line renders only if every one of its fields has a non-empty value.**
   Values are trimmed before the test. If any is empty, the line is dropped whole and
   its keys are omitted from `variables`.
4. **Required fields cannot be empty** — the save is gated — so rule 3 only ever drops
   optional and regular lines.
5. **A non-empty output ends with `\n`.** An output with no kept lines is the empty
   string, not `"\n"`.

Rule 3 is the one to understand: **the line is the unit, not the field.** A line mixing
a filled field and an empty one produces nothing at all, rather than a dangling
fragment like `RM*S-12/`. A template author who wants a field to disappear on its own
MUST put it on its own line.

### Worked example

Template:

```
RM*BOOKINGPAD

# Business type
RM TYPE-{business-type:list(A,B,C)}
RM*OD-{origin:origin}/{destination:destination}
RM*S-{discount:float(2)}/{difference:float(2)}
{{#passengers}}
RM*PAX{n:number}-{ref:travelerReference}/{fare:float(2)}
{{/passengers}}
```

Filled for a WAW→BUD→WAW trip, two passengers, with `difference` left empty:

```
RM*BOOKINGPAD
# Business type
RM TYPE-A
RM*OD-WAW/BUD
RM*PAX1-ADT0/412.35
RM*PAX2-ADT1/389.10
```

The blank line is gone, the comment stayed, the `RM*S-` line vanished entirely because
one of its two fields was empty, and the block produced one line per passenger.

`variables` for that render:

```json
{
  "business-type": "A",
  "origin": "WAW",
  "destination": "BUD",
  "n-ADT0": "1",  "ref-ADT0": "ADT0", "fare-ADT0": "412.35",
  "n-ADT1": "2",  "ref-ADT1": "ADT1", "fare-ADT1": "389.10"
}
```

## The wire shape

A filled remark of either kind serialises to the same object:

| Field | Meaning |
|---|---|
| `id` | id of the template that was filled |
| `template` | **the template string as it stood when the agent filled it** |
| `variables` | flat `label → value` map, per-passenger keys suffixed `-<travelerRef>` |
| `output` | the rendered text |

**`template` is captured at selection time, not re-looked-up.** An agency may edit a
company's template between the booking and a later read; the order MUST still render
what the agent actually filled. A layer that re-resolves `template` from the live
template list is non-conformant.

**`variables` keys are rehydration hints, not identity.** An order retrieve may renumber
or omit traveler references, so a reader matches exact keys first, then fills remaining
per-passenger fields **positionally** from unconsumed keys sharing the same `label-`
prefix. Keys matching nothing are dropped.

Remarks are read and replaced on an order at `GET` / `POST /v2/air/orders/{id}/remarks`.
**The POST is a full replacement** — the complete object, never a partial.

## Mandatory templates

A template may carry two independent flags, `neededOnCreation` and `neededOnIssuance`.
Both default `false`.

- **A company holds at most one remark, full stop.** Not "at most one mandatory
  template within a list" — the company's entire list is capped to one row. See
  [PROFILES.md](PROFILES.md#company-remark-templates). Its one remark may carry
  either flag, both, or neither.
- **An agency holds a list, but at most one remark across that whole list may carry
  either flag.** This is **one combined constraint, not two independent ones**: a
  second remark cannot take `neededOnIssuance` while a different remark already holds
  `neededOnCreation`, and vice versa. There is at most one **enforced template** per
  agency at any time, and if both flags are set they MUST be set on that same one
  remark. A schema that gives each flag its own uniqueness constraint is
  non-conformant — it would let two different remarks each hold one flag, which this
  rule forbids.
- `neededOnCreation` gates order **creation**: both a plain hold and create-and-issue
  refuse to proceed without it filled.
- `neededOnIssuance` gates the **issue** moment specifically: a standalone issue of a
  previously-held order, and the issue step within create-and-issue. It does not block
  a plain hold — an order may sit Pending with `neededOnIssuance` unmet, and only the
  transition to Issued is refused.
- The API refuses setting either flag `true` on a remark other than the agency's
  already-enforced one (when one exists), `409`. A front end MUST disable the control
  rather than let the save fail — offer "require" only on the enforced remark or on a
  list with none yet, and "release" only on the one currently enforced.
- A mandatory template **pins the selection** for whichever transition it gates: the
  agent cannot pick a different one, and that transition cannot proceed until the
  remark's required fields are filled.
- The gate MUST distinguish **"no templates configured"** from **"templates not loaded
  yet"**. A remarks gate that passes while the list is loading is a defect: it lets a
  transition through without a remark the agency requires.
- A mandatory company template applies only once a company is picked for the booking.

## Agency remark templates

Unlike a company, an agency's remark list is **curated and unbounded**: many templates,
ordered, of which at most one may be the agency's enforced one (see *Mandatory
templates* above). Agents pick from this list at booking time; **managing** it — add,
edit, delete, duplicate — is a manager-only action, separate from picking.

| Field | Type | Required | Notes |
|---|---|---|---|
| `id` | UUID | issued | The `public.remarks` primary key, plain — not a minted handle, see [IDS.md](IDS.md). |
| `name` | string(255) | **yes** | Non-empty. What an agent picks from the list. Trimmed. |
| `template` | string | no | The `{placeholder:type}` text, **stored verbatim**. Empty is a legitimate configured state (a name-only placeholder), so it is always returned, as `""`. Cleared with `""`, never `null`. |
| `neededOnCreation` | boolean | no, default `false` | See *Mandatory templates*. |
| `neededOnIssuance` | boolean | no, default `false` | See *Mandatory templates*. |
| `position` | integer | no, default `0` | Display order within the agency's list. |
| `createdAt` / `updatedAt` | timestamp | issued | Server-set. |

| Operation | Path | What it does |
|---|---|---|
| `agencyRemarkList` | `GET /v2/agency/remarks` | Every template on the agent's agency, the enforced one first (if any), then `position`, then name. Not paged — an agency holds, in practice, a handful to a few dozen. |
| `agencyRemarkCreate` | `POST /v2/agency/remarks` | Adds a template. Only `name` is required. **Duplicate** is a client-side affordance, not a separate operation: prefill a create request from an existing template's `name`/`template`, offering neither flag (the enforced-template rule still applies to the copy). |
| `agencyRemarkRetrieve` | `GET /v2/agency/remarks/{remarkId}` | One template as configured on the agency. |
| `agencyRemarkUpdate` | `PATCH /v2/agency/remarks/{remarkId}` | Sparse edit; nothing is nullable. Clearing `name` is refused (`422`). |
| `agencyRemarkDelete` | `DELETE /v2/agency/remarks/{remarkId}` | Detaches the template; hub drops the definition once nothing links to it. Orders already filled in from it keep their own copy of the text. `204`. |

All five sit under `/v2/agency`, session-scoped like `/v2/agency/agents` — no
agency-id path segment; the agency is resolved from the caller's session. Hub's own
`/agw/agencies/{agency_id}/remarks` is the same shape with the agency named explicitly
in the path, since hub has no session to resolve it from.

Read access at booking time is separate and unaffected by this section: the Air
surface's `AirAgencyRemarkTemplates` operation (`GET /v2/air/agency/remark_templates`)
remains the read-only listing an agent picks from while booking; the operations above
are for a manager editing the list itself, e.g. BookingPad's Settings-wheel "Agency
Remark Templates" manager.

## Layer contract

| Layer | Obligation |
|---|---|
| **hub persistence** | Stores `template`, `variables`, `output` as given. Never re-renders, never reformats. Enforces: at most one remark per company; at most one remark per agency carrying either flag (one combined partial unique index, not two). |
| **AGW API V2** | `agencyRemarks` / `companyRemarks` on order create, create-and-issue, and the order remarks resource. Enforces `neededOnCreation` at order creation (hold and create-and-issue) and `neededOnIssuance` at the issue moment (standalone issue, and the issue step of create-and-issue). |
| **BookingPad web** | Owns the parser, the four autofill tokens, and the rendering contract above. Company template authoring lives in Profiles; agency template authoring lives in the Settings-wheel manager, manager-only. |
| **Traveller app** | Never authors or renders remarks. A remark is agency-internal and MUST NOT be shown to a traveller. |
| **Tests** | The rendering contract's five rules each have a test. A change to any of them changes this file first. |

## Known non-conformance

Deliberate, tracked exceptions. They are to be closed, and they are never precedent.

### BookingPad v1 recognises ten autofill tokens this spec does not

BookingPad v1 resolves `psg-index`, `psg-sales-price`, `agent-id`, `agent-custom-id`,
`agency-id`, `order-id`, `account-id`, `ticketnumber`, `ticketserialnumber` and
`airlinecodeticket` in addition to the four canonical tokens. BookingPad v2 does not:
**each becomes a plain text input the agent must type by hand.**

Any agency template using one will silently change behaviour on migration — from
autofilled to agent-typed, with no error. Before an agency moves to v2, its templates
MUST be audited for these ten tokens and either rewritten, or the token promoted into
the canonical table above by PR.

Three of the ten are worth closing outright rather than promoting: `ticketnumber`,
`ticketserialnumber` and `airlinecodeticket` only have values after ticketing, while the
remark form is only editable before it. They cannot be filled in v1 either.

### BookingPad v1's implicit passenger loop

v1 treats any line containing `psg-index` as a passenger loop without delimiters, repeats
**one line only**, and requires the passenger-identifying field to be last on the line.
v2 supports neither the implicit form nor its positional rule, and does support
multi-line blocks. Templates relying on the implicit loop MUST be rewritten with explicit
`{{#passengers}}` delimiters.

### The company template hint contradicts the rendering contract

BookingPad v2's company-remark form tells the agent that a line starting with `#` is
"a note agents see but the airline does not". The renderer keeps comment lines verbatim
in `output`, and its test asserts exactly that. **The rendering contract is correct and
the hint is wrong**; the hint is to be corrected to say the note is sent with the remark.
Until it is, agents may be writing internal notes they believe are private.

### v1's fixed separator between the two kinds

v1 concatenates both kinds into one string separated by a literal
`#------- CORPORATE REMARKS --------` line. v2 keeps them in separate fields, which is
canonical. No layer may reintroduce the separator, and a reader of a v1-era order MUST
treat that line as data, not as structure.
