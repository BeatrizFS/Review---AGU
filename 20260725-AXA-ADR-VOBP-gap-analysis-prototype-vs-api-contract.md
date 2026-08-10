# Gap Analysis — Prototype vs API Contract

| Status | Domain | Author | Date | Version |
|---|---|---|---|---|
| Draft | Integration / Experience | Adilson Arcoverde — Application Architect | 2026-07-25 | 1.3 |

---

## Purpose

This document states, explicitly, **what the prototype requires and the API contract does not
provide**. It decides nothing — the architecture decisions live in
`20260725-AXA-ADR-VOBP-architecture-of-benefit-packages-data-model-backend.md`,
`20260725-AXA-ADR-VOBP-architecture-of-benefits-configuration-endpoints-backend.md` and, for the
signature capability, `20260725-AXA-ADR-VOBP-architecture-of-agreement-signature-backend.md` with its
front-end pair. Its single purpose is to make the unmapped surface visible before development starts,
so that each gap is either closed by a contract change, absorbed by an internal channel, or removed
from the MVP scope by an explicit business decision.

## Scope compared

**Prototype**: Figma file `2yTxzyDRB6IQ9NHkpjklMs`, page `MVP [06.17.2026]`, section `Issuer`
(`803:14439`), comprising:

| Section | Node | Content |
|---|---|---|
| Benefit Packages | `803:14436` | Package list with search, filter, pagination and row actions; package detail (Draft, and Draft without BIN) |
| New package | `803:14437` | The 6-step creation flow, plus the *Without BIN* variant (`1195:20226`) and the benefit detail page (`895:11767`) |
| Confirm action modals | `803:14434` | Duplicate, Disable, Delete |
| Error modals | `803:14435` | Draft couldn't be saved, Package couldn't be created, Something went wrong |
| Filter - open | `1567:8677` | Five filter dimensions |
| Home page | `803:14438` | Landing page, statistics, portfolio, notifications, settings |

**Contract**: the four OpenAPI documents in `specs/vopb/` — `benefit_packages.openapi.yaml`,
`products.openapi 4.yaml`, `banks.openapi.yaml`, `banks__id__agents.openapi.yaml`. Nine
operations in total.

## Summary

**43 gaps**, of which **1 is resolved** and 42 remain open:

| Severity | Count | Meaning |
|---|---|---|
| Blocker | 15 | The screen cannot be rendered or the action cannot be performed with the contract as published. |
| Major | 19 | The screen renders but with degraded behaviour, wrong labels, or a business rule that nothing enforces. |
| Minor | 8 | Cosmetic, or a limit that only bites at scale. |
| Resolved | 1 | Closed by a business decision after this analysis was first issued. |

**G34 moved from blocker to major.** The agreement signature capability
(`20260725-AXA-ADR-VOBP-architecture-of-agreement-signature-backend.md`) closes the part that made it
a blocker — a binding commitment with no server-side evidence. What survives is a transport gap: the
signature exists and is persisted, but the contract still cannot report it.

| Area | Blocker | Major | Minor | Resolved |
|---|---|---|---|---|
| A. Package list, search, filter, pagination | 6 | 4 | 3 | — |
| B. Actions absent from the contract | 3 | 2 | — | — |
| C. Package attributes with no contract field | 3 | 4 | — | 1 |
| D. Catalogue and card-data lookups | 3 | 2 | 2 | — |
| E. Agreement, activation and credits | — | 4 | — | — |
| F. Benefit content | — | 2 | 1 | — |
| G. Home page, notifications, settings | — | 1 | — | — |
| H. Contract defects and inverse gaps | — | — | 2 | — |

**G26** is the resolved item: the BIN is mandatory in the backend, so the *Without BIN* variant of
the prototype is out of scope and the contract's required `affected_policy_range` is correct. The
correction belongs to the design, not to the contract.

The single most consequential finding: **`SimplifiedBenefitPackage`, the schema returned by
`GET /benefit_packages`, is missing four of the nine columns the list screen renders.** The
primary screen of the product cannot be built from its own contract.

---

## A. Package list, search, filter and pagination

Evidence: `213:5856` (list, with the row action menu open) and `1567:8677` (filter panel).

The list renders nine columns: **ID**, **Benefit package name**, **Product**, **BIN**,
**Estimated accounts**, **Category**, **Start date**, **Valid until**, **Status**.
`SimplifiedBenefitPackage` carries six attributes: `benefit_package_id`, `product_reference`,
`active_start_at`, `active_end_at`, `amount_of_policies`, `status`.

| # | Gap | Prototype | Contract | Impact | Severity |
|---|---|---|---|---|---|
| G01 | Package name absent from the list payload | Column *Benefit package name* shows `Benefit Package 1`, `Infinite #1234`, `Gold #9766` | No `name` attribute in `SimplifiedBenefitPackage` — nor anywhere in `benefit_packages.openapi.yaml` | The column cannot be populated. The name is also the primary identifier the agent recognises. | Blocker |
| G02 | Product **name** absent | Column *Product* shows `Gold`, `Infinite`, `Travel money`, `Signature` | Only `product_reference` (`I_C_BR`) is returned | Rendering the column requires resolving every reference against `GET /products` and joining client-side, on every page render. | Blocker |
| G03 | BIN absent | Column *BIN* shows `000000` and the literal `No BIN` | No BIN attribute in either package schema | The column cannot be populated. The `No BIN` value is moot — the BIN is mandatory in the backend (G26) — but the BIN itself is now a required attribute of every package and still has no place in any payload. | Blocker |
| G04 | Category absent | Column *Category* shows `Consumer` / `Commercial` | No portfolio or category attribute | The column cannot be populated. The same value is a filter dimension (G08). | Blocker |
| G05 | Total result count absent | Footer reads `Showing 1 – 10 of 100`, with numbered pages up to `100` and first/last controls | `BenefitPackageList` is a bare array with no envelope, no `total`, no link header | Neither the count nor the page numbers nor the *last page* control can be rendered. The consumer can only detect the end by receiving a short page. | Blocker |
| G06 | Free-text search absent | Search input in the action bar, present on the list and inside the filter panel | `GET /benefit_packages` accepts only `start_index` and `max_results` | Search is impossible server-side. Searching within one page is not the behaviour the screen implies. | Blocker |
| G07 | Filter by product absent | Multi-select with `Gold`, `Infinite`, `Signature`, `Platinum`, plus *Select all* / *Clear all* | No query parameter | Filtering is impossible server-side. | Major |
| G08 | Filter by BIN, category, period and status absent | Four further multi-selects: `All BIN`, `All Categories`, `All Time`, `All Status` | No query parameters | Same as G07, across four more dimensions. Status filtering in particular is how an agent finds drafts. | Major |
| G09 | Column sorting absent | All nine headers are rendered in the link style used for interactive headers | No `sort` or `order_by` parameter | Sorting is either client-side within a page — which is wrong across pages — or absent. | Major |
| G10 | Deep pagination exceeds the platform ceiling | `Results per page` selector (`7` shown) with page `100` reachable | `start_index` has `minimum: 0` and no maximum | SOQL `OFFSET` is capped at 2000. Page 100 at 10 per page is offset 990 and works; at 100 per page it is offset 9900 and cannot be served. The contract permits a request the platform cannot answer. | Major |
| G11 | Status vocabulary mismatch | Badges read `Draft`, `Active`, **`Pending`**, `Expired` | Enum is `DRAFT`, `AWAITING_START`, `ACTIVE`, `CANCELLED`, `EXPIRED` | `Pending` most plausibly means `AWAITING_START`, but that is an inference. The mapping is undocumented. | Minor |
| G12 | `CANCELLED` never rendered | No cancelled badge appears in the list | Enum includes `CANCELLED` | Either cancelled packages are filtered out of the list — which no parameter can express (G08) — or the badge is simply missing from the design. | Minor |
| G13 | Display identifier format | ID column shows `#1234` | `benefit_package_id` is a string of up to 18 characters, example `b09aac28-7f63` | `#1234` is neither. Either a separate human-readable number exists, or the column truncates the technical identifier. | Minor |

---

## B. Actions present in the prototype and absent from the contract

Evidence: row action menu in `213:5856`; confirmation modals in `803:14434`.

The row menu offers **Edit**, **Delete** and **Duplicate**. The modals confirm **Duplicate
package**, **Disable package** and **Delete package**. The contract offers `PATCH`, `POST /cancel`
and `POST /complete`.

| # | Gap | Prototype | Contract | Impact | Severity |
|---|---|---|---|---|---|
| G14 | Delete has no endpoint | *Delete package?* — "This draft will be permanently deleted and cannot be recovered" | No `DELETE` verb on any path | A hard delete of a draft cannot be performed. It is also semantically incompatible with the contract's `CANCELLED` state, which retains the record. | Blocker |
| G15 | Duplicate has no endpoint | *Duplicate package?* — "A copy of this package will be created for you to review and edit" | No duplicate operation | Either the client re-creates the package by reading the source and issuing a `POST` — which silently changes `benefit_package_id`, loses server-side validation of the copy, and cannot copy attributes the contract does not expose (see area C) — or the action is not deliverable. | Blocker |
| G16 | Disable is not the same as cancel | *Disable package?* — "This package and its benefits will no longer be available for accounts" | Only `POST /cancel` exists, whose summary is "validate that he wants to cancel a benefits package" | Three distinct destructive actions in the prototype (Delete a draft, Disable an active package, and whatever *Cancel* means) collapse onto one endpoint. Whether Disable is Cancel under a different label is unresolved, and the answer determines whether an active package's history is preserved. | Blocker |
| G17 | Draft autosave has no distinct operation | Error modal *"Draft couldn't be saved"*, separate from *"Package couldn't be created"* | `POST` creates, `PATCH` updates; nothing distinguishes an incremental draft save | Two different failure messages imply two different operations. If the wizard autosaves per step, each step is a `PATCH` with a partial body — which, under the wholesale-replacement semantics decided in ADR 05, would destroy benefit selections made in an earlier step. | Major |
| G18 | Retry has no idempotency mechanism | Both error modals offer *Try again* | No idempotency key, no `409`, and `POST /complete` and `POST /cancel` are not idempotent | *Try again* after a request that actually succeeded but whose response was lost creates a duplicate package. | Major |

---

## C. Package attributes collected or displayed with no contract field

Evidence: `1131:10133` and `787:9879` (review step), `291:14199` and `1195:19486` (detail),
`1131:10621` (package information step), `1105:7182` (naming step).

| # | Gap | Prototype | Contract | Impact | Severity |
|---|---|---|---|---|---|
| G19 | Package name | *Name your package* step, with the hint "Choose a name that helps you identify this package in the future"; shown on the detail header (`Benefit Package 2`) and in the list | Absent from `BenefitPackage`, `SimplifiedBenefitPackage` and `BenefitPackagePostResponse` | The agent's input has nowhere to go. This is the same gap as G01 seen from the write side, and it affects six of the nine operations. | Blocker |
| G20 | Portfolio | *Cards information* shows `Portfolio: Consumer`; the list calls it *Category* | Absent | Displayed on three screens, filterable on one, and not carried by any payload. **Transport gap only** — the value already exists in the org as `Product2.Card_Platform_Code__c`, one of the three segments of a reference such as `I_C_BR`. | Blocker |
| G21 | Account funding source | *Cards information* shows `Account funding source: Credit`; the home page shows `Infinite - Credit`, `Signature - Debit` | Absent | Same as G20, and likewise a transport gap: the value is `Product2.Card_Type_Code__c`. | Blocker |
| G22 | Renewal type | *Package information* shows `Renewal: Automatic` | Absent | An automatic renewal is a commitment with a financial consequence. Nothing in the contract expresses it, so nothing can be read back to confirm what the agent agreed to. | Major |
| G23 | Billing frequency and billing quarter | `Billing: Quarterly`, `Quarter start date: Quarter 1` | Absent | The billing basis of the package is not readable through the API. | Major |
| G24 | Estimated cost trio | `Total cost per benefit $6.6778 USD`, `Estimated accounts 1,000`, `Estimated total cost $6,677.8 USD`, on the review step and on the detail screen | `BenefitPackage` carries no cost attribute. Prices exist only in `products`, per benefit | The detail screen must recompute the estimate from the catalogue at render time, which means a package's displayed cost changes when the catalogue price changes — even for a package already accepted and activated. | Major |
| G25 | Account range cardinality | The account-selection step renders two to three numeric inputs depending on the BIN variant (`1174:8588` two, `1107:7841` three) | `affected_policy_range` is a single object with exactly two attributes | Whether the flow supports one range or several is not determinable from the design metadata, and the contract admits only one. If the intent is several, the contract shape is wrong, not merely incomplete. | Major |
| G26 | ~~Account range required against a *Without BIN* variant~~ — **resolved by business decision** | `My packages - Details - Draft - No BIN` (`1195:19475`), `Add BIN` (`1195:19882`) and the whole *Without BIN* section (`1195:20226`) | `affected_policy_range` is in the `required` list of `BenefitPackage`; `SimplifiedBenefitPackage` does not carry it at all | The backend requires the BIN to exist, so the BIN and the account range are both mandatory and the contract is right. The correction is on the **prototype**: the *Without BIN* screens describe a state that will not be implemented and must be dropped from the design. See decision 12 of the data model document. | Resolved |

---

## D. Catalogue and card-data lookups with no source

Evidence: `213:9186` (account selection with volume), `1105:7183`, `1195:19882` (Add BIN),
`1567:8677` (filter dimensions).

| # | Gap | Prototype | Contract | Impact | Severity |
|---|---|---|---|---|---|
| G27 | No BIN listing endpoint | The agent picks a BIN; the filter offers an `All BIN` multi-select; *Cards information* displays `BIN 400102` | None of the four contracts return a BIN. `banks/{bank_id}` returns name, country and currency only | The agent cannot be shown the BINs available to their bank. **Transport gap only** — the data exists and is already scoped: `VG_BankIdentificationNumber__c` carries `BIN_Number__c`, `Status__c` and a `Bank__c` lookup, so the bank's approved BINs are one query away. What is missing is an operation to expose them. Raised in severity by G26: the BIN is now mandatory, so the picker is on the critical path of every creation. | Blocker |
| G28 | No accounts-in-range lookup | *Total accounts in this range: 0*, *Estimated total account in this range: 1,000*, and the empty-state copy "There are no cards on this BIN range yet. How many are expected to be by the end of the quarter (billing period)" | No endpoint returns a count of accounts for a BIN or a range | The screen displays a number the platform is never asked for. Note this is a **read** gap and is independent of the decision that `amount_of_policies` is agent-declared — the prototype shows both an observed count and a declared forecast. | Blocker |
| G29 | Products endpoint carries no portfolio, funding source, BIN or currency | The *Cards information* block and three filter dimensions are built from these values | `Product` returns `product_reference`, `name`, `core_benefits`, `optional_benefits` | The product picker can show a name, and nothing else the screens need. | Blocker |
| G30 | Filter option lists have no source | Each filter is a finite multi-select (`Gold`, `Infinite`, `Signature`, `Platinum`; `All Categories`; `All Status`; `All Time`) | Only the product list can be derived, from `GET /products` | Three of five option lists must be hard-coded in the client, which makes them silently wrong when the catalogue changes. | Major |
| G31 | Enrollment window is a rule with no representation | Home page: *Next Enrollment Month — May 2026*, "activation only starts next month or at the beginning of next quarter"; an earlier variant shows a window `04/01/2024 - 04/21/2024`. The package-information step repeats: "You can choose your package start date for next month or any future month" | No endpoint exposes the window, and no validation in the contract constrains `active_start_at` beyond the note that a past date is coerced to today | The rule is enforced only by the client. A direct API call can create a package starting tomorrow, which the business apparently forbids. | Major |
| G32 | Products list truncates silently | — | `ProductsList` has `maxItems: 100` and no pagination parameters | A bank with more than 100 products receives a truncated catalogue with no indication that it is truncated. | Minor |
| G33 | Results-per-page default disagrees | Selector shows `7` | `max_results` defaults to `100` | Cosmetic, but the default page size the API advertises is not the one the screen uses. | Minor |

---

## E. Agreement, activation and credits

Evidence: `277:8092` (agreement), `213:9774` (activation confirmation), `213:9186` (extra cost),
`1131:10148` (review footnote).

| # | Gap | Prototype | Contract | Impact | Severity |
|---|---|---|---|---|---|
| G34 | Agreement acceptance — **partially resolved** | A full *Benefit coverage service agreement* — eight numbered clauses — with "By moving through this agreement and clicking 'I accept,' you acknowledge that you have read, understood, and agreed to be bound by the terms" | No endpoint serves the agreement text, no attribute records acceptance, no attribute carries the accepted version | **Closed on the evidence side, still open on the transport side.** The acceptance is now a DocuSign signature: the document is generated from a template, signed by email, and the instant, signer and template version are persisted on the package, per `20260725-AXA-ADR-VOBP-architecture-of-agreement-signature-backend.md`. Server-side evidence therefore exists. What remains is that the `benefit_packages` contract still carries **no attribute** for acceptance, so a consumer cannot read whether a package is signed, and the agreement text is served by DocuSign rather than by this API. | Major |
| G35 | Activation number is not returned | Confirmation screen shows *Activation Confirmation Text*, *Activation Number*, *Instruction Text* | `POST /benefit_packages` returns only `benefit_package_id`; `POST /complete` returns `204` with no body | The number the confirmation screen displays cannot come from the API. | Major |
| G36 | Credit balance has no endpoint | "You have exceeded your available credits. This extra cost will be added to the next billing", with *Extra cost per account $0.50*, *Total accounts 1,000*, *Total extra cost $500* | Nothing about credits, balances or extra cost | The client cannot know whether credits are exceeded, so the entire block is unrenderable. | Major |
| G37 | Additional-credit contract flow — **covered in design, blocked on scope** | "A contract for acquiring additional credits will be available on the next page", repeated on the review step | No such operation | The signature pipeline now covers it: decision 1 of `20260725-AXA-ADR-VOBP-architecture-of-agreement-signature-backend.md` defines it as a second envelope type with its own template and trigger point. It remains undeliverable because what the signature unlocks depends on credit management, whose scope and owner are unresolved (G36 and R4 of the data model document). The prototype still has no page for it. | Major |

---

## F. Benefit content

Evidence: `895:11782` (benefit detail page), `1107:7841` (core benefit list in the creation flow).

| # | Gap | Prototype | Contract | Impact | Severity |
|---|---|---|---|---|---|
| G38 | Core benefits have no description | The *Core benefits* accordion renders a description under each of the ten benefits | `CoreBenefit` has exactly two attributes, `benefit_id` and `name`, in **both** `benefit_packages` and `products`. Only `OptionalBenefit` has `description` | Ten descriptions are rendered with no source. | Major |
| G39 | Benefit detail page needs eight content blocks the contract cannot carry | Title, summary, an *includes* list of seven items, a *how to use* narrative with assistance phone numbers, a documentation-requirements warning, an insurance disclaimer, a geographic-exclusion clause, and a link to the full terms and conditions | `OptionalBenefit.description` is a single string of up to 2000 characters | The page is structured content; a flat 2000-character string can hold neither the structure nor the volume. The rendered text on that one screen already approaches the limit. | Major |
| G40 | Benefit content is not localised in the contract | — | No language parameter or localised attribute anywhere | Confirmed against the org: `Benefit__c` carries `Benefit_Name_EN__c`, `Benefit_Name_es__c` and `Benefit_Name_pt_BR__c`, but a **single** `Benefit_Description__c`. So the name can be localised and the description cannot — the contract exposes one unnamed language for both. | Minor |

---

## G. Home page, notifications and settings

Evidence: `803:14438` — landing page, statistics, portfolio, notifications, settings.

| # | Gap | Prototype | Contract | Impact | Severity |
|---|---|---|---|---|---|
| G41 | An entire screen has no contract | *Recent benefit packages* (three cards with product, valid-until and total cards), *Statistics* (`Estimated Enrolled Accounts 1,384,873`, `Active packages 12`, `Active cardholders 1,384,873`), *Your portfolio* (per-card package counts: `Card blue 3 packages`), *Notifications*, and *Settings* with `Personal information` and `Communication Preferences` | None of the nine operations serve aggregates, notifications or agent preferences | The landing page — the first screen an agent sees — is entirely outside the contract. *Recent packages* is the only block derivable, by calling the list endpoint and taking the first three; the aggregates cannot be computed client-side without fetching every package. | Major |

The gap is recorded as a single item because it is one scope question, not eight field questions:
whether the home page belongs to this API family at all. Counted once in the summary.

---

## H. Contract defects and inverse gaps

Two defects in the contract, independent of the prototype. Both are the same class — an attribute
declared `readOnly` and `required` in a schema used as a request body:

| # | Defect | Consequence |
|---|---|---|
| G42 | `BenefitPackage.benefit_package_id` is `readOnly: true` and appears in `required` | The client cannot send it and the server cannot demand it, in the schema used by `POST` and `PATCH`. Tracked reciprocally as **R3** of `20260725-AXA-ADR-VOBP-architecture-of-benefit-packages-data-model-backend.md`, whose mitigation is that the POST handler ignores the attribute and never rejects its absence. |
| G43 | `OptionalBenefit.description` is `readOnly: true` and appears in `required` | Same contradiction, on every optional benefit of every write request. A strict validator rejects a well-formed request. Also tracked under **R3** of the data model document, and requested from the API owner in the same correction. |

**Inverse gaps — contract surface the prototype never exercises:**

- `EncryptedPayload` on every operation. Invisible to the UI by nature; no gap, recorded for
  completeness.
- `banks/{bank_id}.base_currency` and `.country`. Nothing in the prototype displays either. The
  currency is nonetheless load-bearing in the data model, because it selects the price book entry.
- `CANCELLED` and `EXPIRED` statuses have no badge in the list design (G12) and no detail screen —
  only `Draft` details exist (`291:14189`, `1195:19475`). The prototype does not show what an
  active, cancelled or expired package looks like.
- `product_reference` as a value is never displayed. The screens show decomposed values instead
  (G20, G21), and the org confirms why: the reference is the composite `Product__c.Key__c`, whose
  segments exist separately on `Product2` as `Card_Platform_Code__c`, `Card_Type_Code__c` and
  `Country_Code__c`. The contract omits the decomposition; the org already has it.

---

## What is explicitly not mapped

Consolidated, in the form a scope decision can be taken against.

**No endpoint exists for:**

1. Deleting a draft package (G14)
2. Duplicating a package (G15)
3. Disabling an active package, if it is distinct from cancelling (G16)
4. Listing the BINs available to a bank (G27)
5. Counting the accounts in a BIN or account range (G28)
6. ~~Retrieving the benefit coverage service agreement text and version~~ — the text now lives in a
   DocuSign template and reaches the signer by email, not through this API (G34)
7. ~~Recording the agent's acceptance of the agreement~~ — **recorded**, as a signature persisted on
   the package; what is still missing is an attribute that lets a consumer *read* it (G34)
8. Reading the credit balance and the extra cost of exceeding it (G36)
9. ~~Acquiring additional credits~~ — the contract envelope is designed; the operation stays blocked
   on credit-management scope (G37)
10. Reading the enrollment window (G31)
11. Home page aggregates: enrolled accounts, active packages, active cardholders, per-card
    portfolio counts (G41)
12. Agent notifications (G41)
13. Agent personal information and communication preferences (G41)
14. Structured benefit content for the benefit detail page (G39)

**No attribute exists for:**

15. Package name (G19, G01)
16. Portfolio / category (G20, G04)
17. Account funding source (G21)
18. BIN on the package (G03)
19. Renewal type (G22)
20. Billing frequency and billing quarter (G23)
21. Estimated total cost, cost per benefit, and the currency of both (G24)
22. Activation number (G35)
23. Agreement acceptance instant, actor and version — now **stored** on the package but still absent
    from every payload, so the transport gap stands (G34)
24. Product name in the list payload (G02)
25. Total result count for pagination (G05)
26. Core benefit description (G38)
27. Content language (G40)

**No query parameter exists for:**

28. Free-text search (G06)
29. Filter by product, BIN, category, period, status (G07, G08)
30. Sorting (G09)
31. Pagination of the products catalogue (G32)

**No rule or mechanism exists for:**

32. Constraining `active_start_at` to the enrollment window (G31)
33. Idempotent retry after a lost response (G18)
34. Incremental draft save distinct from create and from full update (G17)
35. Multiple account ranges per package, if that is the intent (G25)
36. ~~A package legitimately without a BIN and without an account range~~ — **resolved**: the BIN is
    mandatory and the *Without BIN* screens leave the scope (G26)
37. ~~The decomposition rule of `product_reference`~~ — **resolved**: the reference resolves against
    `Product__c.Key__c`, and the decomposed values already exist on `Product2` as
    `Card_Platform_Code__c`, `Card_Type_Code__c` and `Country_Code__c`. No rule to document
    (inverse gap)

---

## Consequences for the architecture already documented

The two architecture documents were written against the contract, with the prototype's extra
attributes modelled but not exposed. This analysis does not invalidate them; it bounds what they
can deliver:

- **ADR 04 already models** package name, portfolio, funding source, BIN, renewal, billing
  frequency, billing quarter, estimated cost, agreement acceptance and activation number
  (decision 8 of that document), so gaps G19 to G24, G34 and G35 are **storage-solved and
  transport-blocked**. Closing them is a contract change, not a model change.
- **ADR 04 does not model** credits (G36, G37), notifications and agent preferences (G41), or
  structured benefit content (G39). Closing those requires new decisions, not just new fields.
- **Two gaps are already answered by the org**, not by a contract change: `product_reference`
  resolves against `Product__c.Key__c` with its segments on `Product2.Card_*` (decision 10 of ADR 04), and
  the BIN is mandatory, which closes G26 against the design (decision 12 of ADR 04).
- **ADR 05 records G05 as R4 and the `OFFSET` ceiling as part of decision 9**; this document adds
  search, filtering and sorting (G06 to G09) to the same defect, which together make the list
  endpoint unfit for the screen it serves.
- **ADR 05 decided wholesale replacement of the benefit collections on `PATCH`** (decision 11).
  G17 shows that decision collides with a per-step autosave: a partial `PATCH` at step 4 would
  clear the selections made at step 3. Either autosave sends the complete accumulated state on
  every call, or the contract needs an incremental operation.
- **Delete, Duplicate and Disable (G14 to G16) have no place in either document**, because no
  contract operation implies them. Nothing in the current architecture can serve those three
  buttons.

---

## Open questions

Business decisions, to be resolved before the contract is frozen:

1. Is *Disable* the same operation as `POST /cancel`, or a distinct state? If distinct, does a
   disabled package keep its billing obligation for the current quarter? (G16)
2. Must a draft be permanently deletable, given that `CANCELLED` retains the record? (G14)
3. Does the account-selection step accept one account range or several? (G25)
4. ~~May a completed package exist without a BIN and without an account range?~~ **Answered: no.**
   The BIN must exist in the backend, so both are mandatory and the *Without BIN* screens
   (`1195:20226`, `1195:19475`, `1195:19882`) leave the scope. (G26)
5. Who owns credit management, and is it in the MVP? (G36, G37)
6. Does the home page belong to this API family, or to a separate aggregation contract? (G41)
7. Is the benefit detail content owned by this family or served from the existing content
   catalogue through another channel? (G39, G40)
8. Is the enrollment window a hard server-side rule or a client-side guidance? (G31)

Questions for the API owner:

9. ~~Confirm the decomposition rule of `product_reference`.~~ **Answered from the org:** the
   reference is `Product__c.Key__c`, and the three segments of `I_C_BR` are already carried by
   `Product2.Card_Platform_Code__c`, `Card_Type_Code__c` and `Country_Code__c`.
10. Confirm the mapping of the `Pending` badge to `AWAITING_START`, or correct the design.
11. Correct the `readOnly` plus `required` contradiction on `benefit_package_id` and on
    `OptionalBenefit.description`. (G42, G43)
12. Confirm whether a total-count envelope, filter parameters and a sort parameter can be added to
    `GET /benefit_packages` in the current version, or whether the list screen must be redesigned
    around what the contract offers.

---

## Next steps

1. Review this document with the product owner and the API owner in a single session, and classify
   each of the 43 gaps as *close in contract v1*, *defer to v2*, or *remove from scope*. The
   16 blockers must all be classified before development of the list screen or the creation flow
   starts.
2. Resolve the 12 open questions above; questions 1 to 8 gate the model, 9 to 12 gate the contract.
3. Reissue the contract with the agreed additions, and update
   `20260725-AXA-ADR-VOBP-architecture-of-benefits-configuration-endpoints-backend.md` to version 1.1
   with the new operations and parameters.
4. Update decision 8 of
   `20260725-AXA-ADR-VOBP-architecture-of-benefit-packages-data-model-backend.md` to move the
   newly exposed attributes out of the not-exposed list.
5. Produce the front-end architecture document only after the blockers are classified — a
   front-end document written against the current contract would specify screens that cannot be
   built.
