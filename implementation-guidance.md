# Deal Sync API v1.1 — Implementation Guidance

This document provides implementation guidance for the Deal Sync API v1.1 specification. It is a companion to [deal1.1.md](deal1.1.md), which defines the object model and API surface. Readers should familiarize themselves with the object definitions before consulting this guide.

## Table of Contents

- [Matching Bid Requests to Deals](#matching-bid-requests-to-deals)
- [Authorization](#authorization)
- [Sending and Receiving Information](#sending-and-receiving-information)
- [Idempotency](#idempotency)
- [Conflict Priority Rules](#conflict-priority-rules)
- [Deal Object Guidance](#deal-object-guidance)
  - [Auxiliary Data](#auxiliary-data)
  - [Publisher Count](#publisher-count)
  - [Dynamic Inventory](#dynamic-inventory)
- [Terms Object](#terms-object)
- [Inventory Object](#inventory-object)
  - [Composition Object Design](#composition-object-design)
  - [Inclusion and Exclusion Semantics](#inclusion-and-exclusion-semantics)
  - [Partial Object Matching](#partial-object-matching)
  - [Field Selection Guidance](#field-selection-guidance)
  - [Relationship to dinventory](#inventory-and-dinventory)
- [Deal Revision Workflow](#deal-revision-workflow)
  - [Revision Semantics](#revision-semantics)
  - [Negotiation and Deal Lifecycle](#negotiation-and-deal-lifecycle)
  - [full_history Query Parameter](#full-history-query-parameter)
  - [Example Workflow](#example-workflow)
- [Price and Floor Guidance](#price-and-floor-guidance)
- [Origin, Curator, and Seller](#origin-curator-and-seller)
  - [Example 1](#example-1)
  - [Example 2](#example-2)
- [Curation Object](#curation-object)
  - [Curation Fee](#curation-fee)
- [Example Scenarios](#example-scenarios)

---

<a name="matching-bid-requests-to-deals"></a>
## Matching Bid Requests to Deals

Some level of trust is required when buying any Deal ID. It is incumbent on the buyer of the deal to compare information from the Deal API with information contained in OpenRTB Bid Requests to ensure that it meets their expectations.

Version 1.1 introduces a deal revision workflow that allows deal terms to be updated after initial send. Implementers should ensure their systems are capable of processing revisions to an existing deal pushed by the counterparty. The operative terms of a deal at any given time are those of the most recently accepted revision (`liverevision`). Buyers should apply targeting based on `liverevision` and should not bid on the basis of a revision that is still PROPOSED. To ensure data integrity, deal receivers are encouraged to implement periodic polling as a fallback mechanism to handle any missed notifications.

The canonical `deal.id` must match the `deal.id` in the OpenRTB bid request when bidding. Both parties should reference this canonical identifier regardless of which party initiated the deal or any party-specific identifiers (`sellerdealid`, `buyerdealid`) maintained in their own systems.

Implementers are strongly encouraged to discuss where targeting criteria will be set. In instances where additional targeting will be applied in the receiving system, implementers should discuss potential implications to delivery if sources of targeting may differ.

<a name="authorization"></a>
## Authorization

Supply Chain validation should always be done using Object: Supply Chain from OpenRTB. If this information is unavailable, implementers should proceed with extreme caution with the understanding that there is no mechanism to know if a given path is authorized to sell the inventory.

<a name="sending-and-receiving-information"></a>
## Sending and Receiving Information

**Baseline (seller-push) model:** At minimum, the seller initiates deals and proposes revisions by pushing Deal objects to the buyer's endpoint (POST), and periodically polls the buyer's endpoint (GET) to retrieve the current state of the deal. The buyer must implement both POST (to receive incoming deals and revisions) and GET (to respond to status queries). In this model, the buyer communicates acceptance or rejection by updating the `negotiationstatus` on `currentrevision` in the Deal object returned via GET. The seller detects the buyer's verdict on its next poll. This model does not require the seller to implement any endpoint.

**Bidirectional model (optional):** When both parties agree to support bidirectionality, either party may initiate a deal or propose a revision by pushing (POST) to the counterparty's endpoint, and both parties implement both POST and GET. In this model, the buyer can also push DealResponse objects directly to the seller's endpoint for faster acceptance/rejection notification, and may initiate deals or propose revisions of their own. The conflict resolution rules described in [Revision Semantics](#revision-semantics) (simultaneous proposals, stale acceptance) apply only when both parties are actively pushing.

**Push endpoint message types:** The push endpoint accepts two message types via HTTP POST:

- **Deal object** — used to create a new deal or propose a revision. The Deal object contains `currentrevision` with the proposed changes.
- **DealResponse object** — used to accept or reject an existing proposed revision. The DealResponse references the target revision by `revisionid` and communicates the verdict via `negotiationstatus`.

Implementations should distinguish between the two based on the payload structure: a DealResponse has a top-level `revisionid` and `negotiationstatus` with no deal term fields; a Deal push has the full Deal object structure. A counter-revision (proposing alternative terms in response to a proposal) is communicated as a new Deal push, not as a DealResponse.

To ensure data integrity, implementers are encouraged to implement periodic polling as a fallback mechanism to handle any missed push notifications.

Implementers may choose to accept incoming webhooks to their API endpoints for events. Please discuss this feature and support with your chosen integration partners.

<a name="idempotency"></a>
## Idempotency

Network failures, timeouts, and retries are inevitable in any distributed system. The Deal Sync API is designed so that push operations can be safely retried without causing unintended side effects. Implementations must treat incoming pushes as idempotent — processing the same message more than once should produce the same result as processing it exactly once.

**Deal pushes:** When a receiver receives a Deal object via POST whose `currentrevision.revisionid` matches a revision it has already processed, it should return an HTTP 200 response containing the current state of the deal. It must not create a duplicate revision, reject the push as an error, or alter the deal's state. The `revisionid` UUID serves as the natural idempotency key — no additional headers or tokens are required.

**DealResponse pushes:** When a receiver receives a DealResponse whose `revisionid` and `negotiationstatus` match a response it has already processed for that revision, it should return an HTTP 200 response with the current deal state. If the referenced `revisionid` has since been superseded by a newer revision, the receiver should still return an HTTP 200 response (not an error) with the current deal state, so that the sender discovers the new `currentrevision` in the same round-trip. See [Conflict Priority Rules](#conflict-priority-rules) for guidance on stale responses.

**Status-only updates:** When a party pushes a Deal object that changes only `sellerstatus` or `buyerstatus` (no new revision), the receiver should apply the update if the new status value represents forward progress in the state transition table. If the incoming status value is the same as or behind the receiver's current record for that field, the receiver should accept the push (HTTP 200) and return the current deal state without modifying it. This makes status pushes naturally idempotent and prevents regressions caused by out-of-order message delivery.

**Recommended receiver behavior summary:**

| Scenario | HTTP Status | Action |
|---|---|---|
| New revision received | 200 | Process normally; store revision |
| Duplicate revision (same `revisionid`, already processed) | 200 | No-op; return current deal state |
| DealResponse for current `currentrevision` | 200 | Process normally; update `negotiationstatus` |
| Duplicate DealResponse (same `revisionid` + `negotiationstatus`, already processed) | 200 | No-op; return current deal state |
| DealResponse for superseded revision | 200 | Discard response; return current deal state (sender discovers new `currentrevision`) |
| Status update representing forward progress | 200 | Apply status change; return current deal state |
| Status update representing same or prior state | 200 | No-op; return current deal state |

All 200 responses should include the current Deal object in the response body. This ensures the sender always receives the most up-to-date view of the deal, regardless of whether their push caused a state change.

<a name="conflict-priority-rules"></a>
## Conflict Priority Rules

When two parties independently act on a deal at the same time — each operating on a potentially stale view of the other's state — conflicting updates can arise. The following rules define how implementations should resolve these conflicts deterministically, ensuring both parties converge to the same state without requiring out-of-band coordination.

**Terminal states dominate.** The terminal states — COMPLETED, EXPIRED, and CANCELED — are irreversible. If a receiver processes an incoming push (whether a revision proposal, a DealResponse, or a status update) and its own record already shows a terminal `sellerstatus` or `buyerstatus`, the receiver should reject the push with an HTTP 409 (Conflict) response containing the current Deal object. The response body gives the sender immediate visibility into the terminal state. Conversely, if an incoming push carries a terminal status, the receiver must apply it even if they have a pending non-terminal action (such as an unprocessed revision proposal). Terminal states always take priority over non-terminal ones, regardless of message ordering.

**Stale revision responses.** A DealResponse must reference the `revisionid` of the revision being acted upon. If the referenced `revisionid` does not match `currentrevision.revisionid` — because the revision has been superseded since the sender last polled — the receiver must discard the DealResponse. The recommended HTTP response is 409 (Conflict) with the current Deal object in the body, allowing the sender to discover the new `currentrevision` and respond to it. Note that the [Idempotency](#idempotency) section takes precedence for one special case: a DealResponse referencing a revision that was already accepted or rejected (same `revisionid` and `negotiationstatus`) should return HTTP 200 as a no-op rather than 409.

**Simultaneous revision proposals.** When both parties push a new PROPOSED revision before receiving the other's push, the seller's revision takes precedence. The buyer's PROPOSED revision transitions to SUPERSEDED, and the seller's revision becomes `currentrevision` on both systems. Both parties apply this rule independently based on the `revisedby.role` field on each revision — because revision identifiers are UUIDs rather than sequential integers, each party can determine whose revision is whose and converge to the same outcome without coordination. See [Revision Semantics](#revision-semantics) for the full rule.

**Recommended error response summary:**

| Conflict scenario | HTTP Status | Response body | Sender action |
|---|---|---|---|
| Push received while deal is in terminal state | 409 | Current Deal object (showing terminal status) | Acknowledge terminal state; stop further updates |
| DealResponse references superseded `revisionid` | 409 | Current Deal object (showing new `currentrevision`) | Evaluate new `currentrevision` and respond to it |
| Simultaneous revision proposals (buyer's revision loses) | 200 | Current Deal object (showing seller's revision as `currentrevision`, buyer's as SUPERSEDED) | Evaluate seller's `currentrevision` |

**Consistency through polling.** Even with these deterministic rules, implementations should not rely solely on push delivery for correctness. Periodic polling (GET) serves as the consistency backstop: if a push is lost, delayed, or its response is not received, both parties will converge on the next poll cycle. Implementations should poll at a reasonable interval (suggested: at least once every 15 minutes for active deals) to bound the window of inconsistency.

---

<a name="deal-object-guidance"></a>
## Deal Object Guidance

<a name="auxiliary-data"></a>
### Auxiliary Data

`auxdata` should be used whenever any non-publisher data is expected to be part of a given deal. This will most often be traditional data providers, but can include optimization services that do NOT belong to the publisher. While the non-publisher data entities are not named, if any company is being paid for their data and/or optimization, this value should indicate the presence of non-publisher data.

This attribute also communicates information about whether or not additional data layers may be applied after the deal has gone live. For example, a deal starts with some additional data and expects the list of data and/or optimization services to be added or updated as the deal is in flight should use a value of 2.

<a name="publisher-count"></a>
### Publisher Count

`pubcount` is meant to communicate if the deal is for a single publishing company, or if it spans multiple publishing companies. Deals covering a single company with multiple web and app properties are considered to be a single publisher. Inventory aggregation services representing multiple publishing sites may also be considered a single publisher deal.

Note that the `pubcount` attribute does **not** contemplate non-publisher parties, including ad tech stacks involved in the transaction. Intermediaries should be identified using Supply Chain Validation from OpenRTB requests.

<a name="dynamic-inventory"></a>
### Dynamic Inventory

`dinventory` communicates information about whether or not the sites and apps included in the initial offering are expected to be updated after the deal goes live. In other words, this attribute communicates the expectation that inventory will (or will not) change by adding or removing sites and/or apps after the deal has gone live. Implementers are strongly encouraged to communicate and discuss the specifics of any and all inventory updates to ensure expectations are met.

<a name="terms-object"></a>
## Terms Object

Price attribute must be provided if the deal is a fixed price deal, especially if it's a programmatic guaranteed deal, but otherwise can be used as pricing guidance or not provided at all if there is no pricing guidance (for example, for a Run-of-Exchange deal where there is no pricing guidance).

Where multiple DSP seats are included, per seat acceptance/rejection is on the DSP to surface in their internal systems and User Interfaces.

<a name="inventory-object"></a>
## Inventory Object

Composition sub-objects should only include dimensions that are shared across all bid requests associated with the deal. For static inventory deals (`dinventory=1`), all five composition dimensions may be used. For dynamic inventory deals (`dinventory=2`), the non-site/app composition dimensions (`contentcomp`, `devicecomp`, `usercomp`) remain meaningful and are encouraged, as they describe the profile of the supply rather than enumerating specific properties. `sitecomp` and `appcomp` may also be included for dynamic deals — for example, to identify publishers and support advance supply authorization — but the seller should keep them current via the revision workflow as inventory changes. See [Relationship to dinventory](#inventory-and-dinventory) for the full interaction guidance.

<a name="composition-object-design"></a>
### Composition Object Design

The v1.1 Inventory object replaces the flat attribute model from v1.0 with a composition-based model aligned to OpenRTB 2.6. Five sub-objects — Content, Device, Audience, Site, and App — each use `incl`/`excl` arrays of their corresponding OpenRTB 2.6 objects to express inventory profiles at any level of granularity. The exception is `usercomp`, which uses OpenRTB 2.6 **Data** objects (the `user.data` sub-structure) rather than full User objects, consistent with the IAB Tech Lab Curated Audiences standard. See [Object: UserComposition](deal1.1.md#object-usercomposition) for details and refer to the Curated Audiences specification for `segtax` taxonomy enumeration.

<a name="inclusion-and-exclusion-semantics"></a>
### Inclusion and Exclusion Semantics

Inventory must match at least one `incl` entry to qualify; matching any `excl` entry disqualifies it regardless of inclusion matches (exclusions take precedence). An absent or empty `incl` array imposes no inclusion constraint; an omitted sub-object imposes no constraint along that dimension.

<a name="partial-object-matching"></a>
### Partial Object Matching

Each entry in an `incl` or `excl` array is a partial or fully specified OpenRTB 2.6 object. Only the fields that are explicitly present in a given entry are used as matching criteria. Fields that are absent from an entry are treated as wildcards — they match any value.

For example, a Content object entry specifying only `{"genre": "sports"}` would match any bid request content object whose genre is sports, regardless of any other content attributes. An entry specifying `{"genre": "sports", "livestream": 1}` would match only sports content that is also signaled as a live broadcast.

Implementers are encouraged to keep composition entries as concise as possible, specifying only the fields that are necessary to accurately describe the intended inventory profile. Overly broad or overly narrow entries may lead to unexpected matching behavior and should be validated against representative bid request samples before the deal goes live.

<a name="field-selection-guidance"></a>
### Field Selection Guidance

Composition objects should only list dimensions that are shared across all bid requests associated with the deal. When deciding which fields to populate in a composition entry, the guiding question is: "Is this characteristic expected to hold across all impression opportunities in this deal?" If so, it is a good candidate. If the value is transient, per-impression, or user-session specific, it is not appropriate here. Buyers should expect that all impressions delivered on the deal will match the specified dimensions.

The following guidance applies to each composition sub-object.

**ContentComposition**

Content fields should reflect attributes that are common to all content included in the deal. For example, a deal covering all episodes of a particular television series should specify the `series` field on a single Content object entry rather than listing each `episode` individually. If a deal is genuinely scoped to a specific, enumerated list of episodes, then listing each `episode` explicitly as a separate `incl` entry is appropriate. Fields such as `genre`, `cat`, `language`, `contentrating`, and `prodq` are well-suited to characterize the content environment at a supply level. Highly dynamic or impression-specific metadata — such as `url` — is generally not appropriate.

**DeviceComposition**

Device fields should describe stable hardware, software, or geographic characteristics of the supply. Fields such as `devicetype`, `make`, `os`, `osv`, and `geo` are appropriate because they are intrinsic properties of the device or its general location that hold across the flight of the deal. By contrast, `ip` and `ipv6` are not appropriate: they are transient per-impression signals that change with each bid request and cannot meaningfully characterize the supply. If the intent is to target impressions associated with a particular IP range or geographic cluster, that intent should be expressed as an audience using the `usercomp` object.

**UserComposition**

The `usercomp` object is designed to express audience characteristics of the deal's supply using anonymized, taxonomy-based cohort signals. Curated audience taxonomy entries (Data objects with `name`, `ext.segtax`, and `segment` arrays) are the primary intended use. Broad geographic signals, if conveyed here rather than in `devicecomp`, are also appropriate.

User-specific or session-specific identifiers are not appropriate in `usercomp` entries. In particular, `buyeruid` and Extended IDs (`eids`) identify individual users and have no place as supply-level composition signals. Their inclusion here would misrepresent the nature of the object and undermine the privacy principles of the Curated Audiences standard.

**SiteComposition and AppComposition**

It is strongly recommended that `incl` entries within `sitecomp` and `appcomp` include the `publisher` object (with at minimum `publisher.domain`) and the `inventorypartnerdomain` field. These fields enable buyers to perform supply chain authorization checks — cross-referencing the authorized seller entries in the relevant ads.txt and app-ads.txt records — in advance of the deal flight, without relying solely on bid request validation once the deal is live. The `domain` field (for sites) and `bundle` field (for apps) are well-suited for identifying specific properties; the `cat` field can characterize content category at the property level.

<a name="inventory-and-dinventory"></a>
### Relationship to `dinventory`

The `dinventory` field on the Deal object is a forward-looking statement about **temporal stability**: will the set of sites and applications included in this deal change after it goes live? Because composition sub-objects should only list dimensions shared across all bid requests, `dinventory` determines how that guarantee interacts with change over time.

For static deals (`dinventory=1`), the composition is a complete and stable picture of the supply — it will not change over the flight. This gives buyers the highest confidence for advance targeting and validation.

For dynamic deals (`dinventory=2`), the composition describes the supply as of the time the deal was sent, but the underlying inventory may evolve. Sellers using dynamic inventory are obligated to push composition updates via the revision workflow whenever the supply changes materially — otherwise the composition becomes stale.

**Including the Inventory object for dynamic deals**

Prior to v1.1, the Inventory object was restricted to `dinventory=1` because it consisted of specific site and app lists that only made sense for fixed supply. The v1.1 composition model changes this. Dimensions such as `contentcomp`, `devicecomp`, and `usercomp` describe the *nature and profile* of the supply rather than enumerating specific properties — and that profile is meaningful and useful regardless of whether the supply is static or dynamic. Sellers are encouraged to include these dimensions for dynamic deals to give buyers useful advance signal about the content environments, device types, and audience segments they should expect.

For `sitecomp` and `appcomp` in a dynamic deal, the composition can still be valuable — for example, to identify the publisher and support supply authorization checks — but the seller should keep them current via the revision workflow as inventory changes.

<a name="deal-revision-workflow"></a>
## Deal Revision Workflow

Version 1.1 introduces a structured mechanism for sellers and buyers to propose and negotiate changes to deal terms after initial deal creation. The revision workflow is built on four fields on the Deal object (`currentrevision`, `liverevision`, `sellerstatus`, `buyerstatus`) and two new objects (`DealRevision`, `DealActor`).

<a name="revision-semantics"></a>
### Revision Semantics

Each deal begins with an initial revision created by the initiating party when the deal is first pushed. Subsequent revisions are created whenever either party proposes a change to deal terms.

**Implementation models:** The revision workflow operates under the same baseline (seller-push) and optional bidirectional models described in [Sending and Receiving Information](#sending-and-receiving-information). The conflict resolution rules below (simultaneous proposals, stale acceptance) apply only when both parties are actively pushing.

**Delta relative to liverevision:** Once a `liverevision` exists, each DealRevision's fields represent a delta relative to it — not relative to the immediately preceding proposed revision. Only fields whose values differ from `liverevision` need be included; fields absent from a `DealRevision` imply no change along that dimension. This design ensures that the meaning of any pending revision is always self-contained and unambiguous, regardless of the chain of proposals that preceded it. Before any revision has been accepted (i.e., while `liverevision` is absent), every proposed revision must be a full specification of the deal terms, since there is no baseline to compute a delta against.

**One pending revision at a time:** At most one revision may be in PROPOSED state (`negotiationstatus=0`) at any time. If a new revision is created while an existing revision is still PROPOSED, the existing revision is automatically transitioned to SUPERSEDED (`negotiationstatus=3`) before the new revision is recorded. A party may only act on `currentrevision`; a revision with `negotiationstatus=3` is non-actionable.

**Simultaneous proposals (conflict resolution):** Because either party may initiate a revision, both may push a new PROPOSED revision before receiving the other's push. When a party receives an incoming PROPOSED revision while they also have an outstanding PROPOSED revision, the seller's revision takes precedence: the buyer's PROPOSED revision transitions to SUPERSEDED, and the seller's revision becomes `currentrevision` on both systems. Both parties independently apply this rule — because UUIDs are used as revision identifiers rather than sequential integers, each party can identify which revision is whose and converge to the same outcome without coordination.

**Stale acceptance and rejection:** Any DealResponse pushed by a party must reference the `revisionid` of the revision being acted upon. Upon receiving a DealResponse, the receiving party must validate that the referenced `revisionid` matches their current `currentrevision.revisionid`. If the IDs do not match — because the revision has since been superseded — the DealResponse is stale and must be discarded. The responding party should re-evaluate `currentrevision` and respond to the correct revision. See [Object: DealResponse](deal1.1.md#object-dealresponse) for the message format.

<a name="negotiation-and-deal-lifecycle"></a>
### Negotiation and Deal Lifecycle

**`negotiationstatus`** (on DealRevision) tracks the per-revision negotiation outcome:

- `0` PROPOSED: The revision has been submitted and awaits the counterparty's response.
- `1` ACCEPTED: The revision has been accepted. This revision becomes `liverevision` and its terms are now operative.
- `2` REJECTED: The revision was rejected. The prior `liverevision` (if any) remains the operative state.
- `3` SUPERSEDED: The revision was replaced by a newer revision before the counterparty could act.

**`sellerstatus`** (on Deal) tracks the lifecycle of the deal from the seller's perspective, independent of any individual revision's negotiation outcome:

- `0` PENDING: Deal has been pushed, awaiting counterparty acceptance.
- `1` NOT_STARTED: Deal has been accepted but the flight start date has not yet been reached.
- `2` LIVE: Deal is active and the seller is trafficking against it.
- `3` LIVE_NOT_SPENDING: Deal is live but no spend has been observed within an expected window (seller diagnostic).
- `4` PAUSED: Seller has temporarily suspended the deal.
- `5` COMPLETED: Deal has reached its end date or delivery goal.
- `6` EXPIRED: Deal lapsed without being activated.
- `7` CANCELED: Deal was terminated prior to completion.

**`buyerstatus`** (on Deal) tracks the lifecycle of the deal from the buyer's perspective:

- `0` PENDING: Deal has been received, awaiting buyer review or acceptance.
- `1` NOT_STARTED: Deal has been accepted but the flight start date has not yet been reached.
- `2` LIVE: Buyer is actively trafficking against the deal.
- `3` PAUSED: Buyer has temporarily suspended their use of the deal.
- `4` COMPLETED: Deal has reached its end date or delivery goal.
- `5` EXPIRED: Deal lapsed without being activated.
- `6` CANCELED: Deal was terminated prior to completion.

**`sellerstatus` transitions:** The following table defines valid transitions from the seller's perspective. Three states — COMPLETED, EXPIRED, and CANCELED — are terminal; no outgoing transitions are permitted from them. To reactivate a canceled deal, the parties should create a new deal.

| From | To | Trigger | Initiated by |
|---|---|---|---|
| PENDING (0) | NOT_STARTED (1) | Buyer accepts initial revision | Automatic (revision workflow) |
| PENDING (0) | CANCELED (7) | Either party withdraws before acceptance | Either party |
| NOT_STARTED (1) | LIVE (2) | Flight date begins or seller activates | Seller or automatic |
| NOT_STARTED (1) | EXPIRED (6) | End date passes before activation | Automatic |
| NOT_STARTED (1) | CANCELED (7) | Either party cancels before deal goes live | Either party |
| LIVE (2) | LIVE_NOT_SPENDING (3) | No spend observed during active flight | Seller (diagnostic) |
| LIVE (2) | PAUSED (4) | Seller pauses the deal | Seller |
| LIVE (2) | COMPLETED (5) | Flight ends or delivery goal met | Automatic |
| LIVE (2) | EXPIRED (6) | End date passes | Automatic |
| LIVE (2) | CANCELED (7) | Either party cancels mid-flight | Either party |
| LIVE_NOT_SPENDING (3) | LIVE (2) | Spending resumes | Seller (diagnostic) |
| LIVE_NOT_SPENDING (3) | PAUSED (4) | Seller pauses the deal | Seller |
| LIVE_NOT_SPENDING (3) | COMPLETED (5) | Flight ends | Automatic |
| LIVE_NOT_SPENDING (3) | EXPIRED (6) | End date passes | Automatic |
| LIVE_NOT_SPENDING (3) | CANCELED (7) | Either party cancels | Either party |
| PAUSED (4) | LIVE (2) | Seller resumes the deal | Seller |
| PAUSED (4) | COMPLETED (5) | Flight ends while paused | Automatic |
| PAUSED (4) | EXPIRED (6) | End date passes while paused | Automatic |
| PAUSED (4) | CANCELED (7) | Either party cancels while paused | Either party |

**`buyerstatus` transitions:** The following table defines valid transitions from the buyer's perspective. COMPLETED, EXPIRED, and CANCELED are terminal.

| From | To | Trigger | Initiated by |
|---|---|---|---|
| PENDING (0) | NOT_STARTED (1) | Buyer accepts initial revision | Buyer (DealResponse or internal acceptance) |
| PENDING (0) | CANCELED (6) | Either party withdraws before acceptance | Either party |
| NOT_STARTED (1) | LIVE (2) | Flight date begins | Automatic |
| NOT_STARTED (1) | EXPIRED (5) | End date passes before activation | Automatic |
| NOT_STARTED (1) | CANCELED (6) | Either party cancels before deal goes live | Either party |
| LIVE (2) | PAUSED (3) | Buyer pauses their use of the deal | Buyer |
| LIVE (2) | COMPLETED (4) | Flight ends or delivery goal met | Automatic |
| LIVE (2) | EXPIRED (5) | End date passes | Automatic |
| LIVE (2) | CANCELED (6) | Either party cancels mid-flight | Either party |
| PAUSED (3) | LIVE (2) | Buyer resumes their use of the deal | Buyer |
| PAUSED (3) | COMPLETED (4) | Flight ends while paused | Automatic |
| PAUSED (3) | EXPIRED (5) | End date passes while paused | Automatic |
| PAUSED (3) | CANCELED (6) | Either party cancels while paused | Either party |

**Coordination rules:** `sellerstatus` and `buyerstatus` are each party's own operational view and may differ legitimately — a buyer pause does not imply a seller pause and vice versa. However, terminal states (COMPLETED, EXPIRED, CANCELED) should be reflected symmetrically: when one party sets a terminal state, the counterparty should mirror it upon learning of it (via push or poll). Note that `sellerstatus=PAUSED` is the effective deal-dark signal: since the seller controls bid request delivery, a seller pause suppresses auction eligibility regardless of `buyerstatus`. Each party populates only their own status field when pushing a Deal object; a party may additionally echo the counterparty's last-known status if they have learned it via polling.

Proposing a revision on a deal that is already LIVE, PAUSED, or LIVE_NOT_SPENDING does **not** change `sellerstatus` or `buyerstatus`. The deal continues to operate under `liverevision` terms while `currentrevision` is pending. `negotiationstatus` on `currentrevision` independently tracks whether a proposed change has been accepted.

**Relationship to `SeatStatus.status`:** The `buyerstatus` field on the Deal object and the `status` field on the SeatStatus object serve different purposes and operate at different levels of granularity. `buyerstatus` reflects the buyer's deal-level lifecycle view — has the buyer accepted, are they trafficking, have they paused? `SeatStatus.status` reflects the per-seat operational state within the buyer's system — has the trader approved it, is it in a campaign, is it actively spending? These two statuses are independent. A deal may have `buyerstatus=2` (LIVE) while a particular buyer seat has `SeatStatus.status=5` (paused) — this is not contradictory; it means the buyer is trafficking the deal but that specific seat has paused its campaign against it. Implementers should not attempt to reconcile these statuses into a single value.

<a name="full-history-query-parameter"></a>
### `full_history` Query Parameter

By default, the Deal API returns only `currentrevision` and `liverevision` on the Deal object. When the counterparty's endpoint supports it, either party may append `full_history=1` to the GET query parameters to request the complete ordered array of all prior revisions for the deal. This allows either party to audit the full negotiation history — including all PROPOSED, REJECTED, and SUPERSEDED revisions — when needed for dispute resolution or deal analysis. Implementations are not required to retain or serve full revision history, but are encouraged to do so.

<a name="example-workflow"></a>
### Example Workflow

The following illustrates a typical revision lifecycle. Revision labels (Revision 1, Revision 2, etc.) are used for readability; in the actual data model each revision is identified by its UUID `revisionid`.

1. **Seller creates the deal.** Seller pushes a Deal object with `currentrevision` set to a new DealRevision (`revisionid=<uuid-A>`, `negotiationstatus=0` PROPOSED, `revisedby.role=0` SELLER). `liverevision` is absent. `sellerstatus=0` (PENDING). Buyer's system initializes `buyerstatus=0` (PENDING) upon receiving the push.

2. **Buyer accepts.** The buyer pushes a DealResponse with `dealid=<deal-id>`, `revisionid=<uuid-A>`, `negotiationstatus=1` (ACCEPTED), and `respondedby.role=1` (BUYER). The seller validates that `<uuid-A>` matches `currentrevision.revisionid`. `negotiationstatus` on Revision 1 transitions to `1` (ACCEPTED). `liverevision` is set to Revision 1. Both `sellerstatus` and `buyerstatus` transition to `1` (NOT_STARTED) or `2` (LIVE) depending on flight dates.

3. **Seller proposes a price change.** Seller creates a new DealRevision (`revisionid=<uuid-B>`, `negotiationstatus=0` PROPOSED, `revisedby.role=0` SELLER) with a `terms` delta containing only the fields that changed from Revision 1. `currentrevision` points to Revision 2. `liverevision` still points to Revision 1.

4. **Seller supersedes their own proposal.** Before the buyer responds, the seller creates Revision 3 (`revisionid=<uuid-C>`). Revision 2 (`<uuid-B>`) transitions to SUPERSEDED. `currentrevision` points to Revision 3. Any acceptance referencing `<uuid-B>` would be rejected as stale.

5. **Buyer rejects Revision 3.** Buyer pushes a DealResponse with `revisionid=<uuid-C>` and `negotiationstatus=2` (REJECTED). `negotiationstatus` on Revision 3 transitions to `2` (REJECTED). `liverevision` remains Revision 1.

6. **Buyer proposes a counter-offer.** Buyer creates Revision 4 (`revisionid=<uuid-D>`, `revisedby.role=1` BUYER) and pushes it to the seller's push endpoint. `currentrevision` points to Revision 4. `negotiationstatus=0` (PROPOSED).

7. **Seller accepts.** Seller pushes a DealResponse with `revisionid=<uuid-D>` and `negotiationstatus=1` (ACCEPTED). `negotiationstatus` on Revision 4 transitions to `1` (ACCEPTED). `liverevision` is updated to Revision 4. The deal's operative terms are now those of Revision 4 relative to Revision 1.

<a name="price-and-floor-guidance"></a>
## Price and Floor Guidance

Historically deals have been negotiated at agreed upon rates to guarantee delivery at a certain price. In the new world many SSPs and curators aggregate media around data points that are not directly related to the inventory itself. Impressions for deals thus curated could have varying price points on a request by request basis. In lieu of a floor provided in these cases, suppliers offer bid guidance that will provide an effective win rate across inventory but do not describe the individual floors of the bid requests included in the deal.

<a name="origin-curator-and-seller"></a>
## Origin, Curator, and Seller

The `origin` attribute identifies the advertising system that will receive bid responses for the deal — typically the SSP running the auction. This field identifies the auction operator, not necessarily the party who initiated the deal. In a buyer-initiated deal, `origin` still refers to the SSP that will conduct the auction.

The `curator` attribute names the business entity that did the packaging of inventory, technology and/or data. Most often, although not always, this will be the party that sold the deal to the buyer.

The `seller` attribute represents the business entity who has sourced the demand (aka contracted with the buyer of the deal). When this is NOT either of the entities named in the `origin` or `curator` attributes, this field can be especially valuable.

<a name="example-1"></a>
### Example 1

- SALESHOUSE has the sales team and sources the demand
- The curated deal incorporates data provided by DATA PROVIDER A, DATA PROVIDER B, and potentially DATA PROVIDER C and/or DATA PROVIDER D
- The deal is entered into the SSP
- CURATOR does all the ops to help SALESHOUSE launch the deal

In this case the `seller`=saleshousedomain.com, the `curator`=curatordomain.com, and the `origin`=sspdomain.com

<a name="example-2"></a>
### Example 2

- PUBLISHER has an agreement with DATA PROVIDER A to decorate bid requests and packages the publisher's inventory
- PUBLISHER sources demand for the deal
- PUBLISHER inputs the deal into SSP User interface

In this case the `seller`=publishercompany.com, the `curator`=datacompany.com, and the `origin`=sspdomain.com

<a name="curation-object"></a>
## Curation Object

<a name="curation-fee"></a>
### Curation Fee

`curfeetype` provides information about the type of fee being applied in the bidstream, but not what that fee is. For example, if a curator is charging a CPM of $5, the `curfeetype` will equal 3 because it is a CPM. If a curator is charging a flat fee of $100, the value sent in the `curfeetype` attribute will be 2, because it is a flat fee. If the curator is packaging their data alongside the inventory and not taking a specific fee for the Curation service itself, the value will be 4 for no fee.

<a name="example-scenarios"></a>
## Example Scenarios

| Scenario | curfeetype | auxdata | pubcount | dinventory |
|----------|:-----------:|:-------:|:--------:|:----------:|
| Publisher packages their O&O inventory and data, and sells a deal to a buyer. No other party is or will be involved in the deal and the inventory will remain static throughout the term of the deal. | 4 | 3 | 1 | 1 |
| Data company packages their data across multiple publishers, but does not charge a fee for the specific curation service. Data providers will not be changed, but inventory may be updated after the start of the deal. | 4 | 1 | 2 | 2 |
| Curation company charges a CPM fee to aggregate inventory across multiple publishers that they will add and remove as the deal is in flight to optimize deal performance. They have an optimization service, and work with additional data providers to increase addressability, and expect to add additional data partners throughout the flight. | 3 | 2 | 2 | 2 |
| Curation company works with inventory aggregation company that regularly adds new websites and charges a % of spend to optimize deals. They do not currently work with other data providers, but may layer them in based on deal performance. | 1 | 4 | 1 | 2 |
| SSP pays a Curator a CPM to decorate inventory across the SSP universe that they do not pass on to the publishers. All inventory and data is subject to change. | 3 | 2 | 2 | 2 |

---
