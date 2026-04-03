# Deal Sync API Specification
**Version 1.1**

## Table of Contents

- [Introduction](#introduction)
  - [Goals](#goals)
  - [What it is](#what-it-is)
  - [What it isn't](#what-it-isnt)
  - [Considerations](#considerations)
  - [Changes in Version 1.1](#changes-in-version-11)
- [Deal API Specification](#deal-api-specification)
  - [Object: Deal](#object-deal)
  - [Object: Terms](#object-terms)
  - [Object: Inventory](#object-inventory)
    - [Object: ContentComposition](#object-contentcomposition)
    - [Object: DeviceComposition](#object-devicecomposition)
    - [Object: UserComposition](#object-usercomposition)
    - [Object: SiteComposition](#object-sitecomposition)
    - [Object: AppComposition](#object-appcomposition)
  - [Object: Curation](#object-curation)
  - [Object: DealRevision](#object-dealrevision)
  - [Object: DealActor](#object-dealactor)
  - [Object: DealResponse](#object-dealresponse)
  - [Object: SeatStatus](#object-seatstatus)
- [Implementation Guidance](#implementation-guidance) *(see [implementation-guidance.md](implementation-guidance.md))*

---

<a name="introduction"></a>
# Introduction

This API has multiple implications including lowering manual entry by making the high-level terms of the deal clear so automated setups can be more easily executed. It also adds transparency into Curated deals that is not currently available in the bid stream.

<a name="goals"></a>
## Goals

- Decrease manual entry of deal information across systems by providing a way for the terms of a deal to be input and sent to the system that will deliver the deal to review and accept.
- Describe what the selling system deal includes at a high level
- Know which parties were involved in curating and selling the package

<a name="what-it-is"></a>
## What it is

- API that provides subscribers with static information about a given Deal that outlines the tenets of the Deal.
- This API supports a bidirectional push model. In its simplest form, the seller pushes deal information to the buyer's endpoint (the traditional seller-push model from v1.0). When both parties agree to support bidirectionality, either party — seller (e.g., SSP) or buyer (e.g., DSP) — may initiate a deal or propose a revision by pushing to the other party's endpoint. A seller-push-only implementation is fully compliant with this specification; bidirectional support is optional and requires mutual agreement between counterparties.
- Version 1.1 introduces a deal revision workflow that enables sellers and buyers to propose, accept, or reject changes to deal terms over the flight of the deal. All updates to deal terms — including changes to inventory, pricing, and other metadata — flow through this revision workflow. To ensure data integrity, deal receivers are encouraged to implement periodic polling as a fallback mechanism to handle any missed notifications.

<a name="what-it-isnt"></a>
## What it isn't

- This is a separate API that is **NOT** included in OpenRTB Request/Response
- Does not include real-time information contained in the bid request or determine whether or not a deal applies. Implementers are strongly encouraged to validate the conditions laid out in this deal with OpenRTB requests to ensure their expectations are being met.
- This API is not a deal discovery marketplace — it does not support browsing, searching, or finding available inventory packages. While v1.1 introduces structured proposal and acceptance of deal revisions between known counterparties, it is not designed for synchronous or real-time interactive negotiation.

<a name="considerations"></a>
## Considerations

- The values of these fields are meant to be an outline of business terms agreed to between buyers and sellers *a priori*. The information contained in this API should be used as a guide of what the terms that were agreed to, but they should never be used to bid without validating that information coming from the OpenRTB Bid Request is in line with the terms laid out in this API.
- Deal criteria provided via this channel are not guaranteed to be accurately reflected in bid requests containing the associated deal ID. It is the responsibility of the buyer to validate bid requests that contain the deal ID conform to the deal criteria. In other words, buyers should ALWAYS apply targeting matching the terms outlined in any deal on their side. For example, if a deal is struck for US only inventory, buyers should include that targeting in their DSP when trafficking the campaign.
- If bid requests continually do not match the expectations laid out in the API, a conversation should be had between the buyer and seller to continue negotiations offline.
- It is not possible to validate that the party selling the inventory has been authorized without including app bundle or site id that matches the publishers (app)ads.txt file. Deal IDs receiving requests without that information should be used rarely and closely monitored.

<a name="changes-in-version-11"></a>
## Changes in Version 1.1

Version 1.1 introduces the following changes from Version 1.0:

- **Inventory Composition Model:** The `Inventory` object has been redesigned to support a richer, composition-based approach to describing inventory. The flat attribute model from v1.0 is replaced by five composition sub-objects — `ContentComposition`, `DeviceComposition`, `UserComposition`, `SiteComposition`, and `AppComposition` — each supporting explicit inclusion and exclusion arrays, enabling deals to be described with greater precision and expressiveness. Four sub-objects (`contentcomp`, `devicecomp`, `sitecomp`, `appcomp`) use arrays of their corresponding OpenRTB 2.6 top-level objects. `UserComposition` uses arrays of OpenRTB 2.6 Data objects (the `user.data` structure), scoped to curated audience cohort signals per the IAB Tech Lab Curated Audiences standard.
- **Bidirectional API Model:** Version 1.1 extends the API to support an optional bidirectional push model. Either party — seller (e.g., SSP) or buyer (e.g., DSP) — may now initiate a deal or propose a revision by pushing to the other party's endpoint, provided both parties have agreed to support bidirectional communication. A seller-push-only implementation remains fully compliant. The `Deal` object gains `sellerdealid` and `buyerdealid` fields to allow each party to maintain their own namespace identifier for a deal alongside the canonical bid-stream `id`.
- **Deal Revision Workflow:** Version 1.1 introduces a structured revision lifecycle for deal terms, replacing the need for ad-hoc differential overrides. All updates to deal terms — whether minor adjustments or substantive renegotiations — flow through the same revision workflow: a party proposes a revision, and the counterparty accepts or rejects it. The `Deal` object gains `currentrevision`, `liverevision`, `sellerstatus`, and `buyerstatus` fields. The new `DealRevision`, `DealActor`, and `DealResponse` objects provide a standardized mechanism for either party to propose, accept, or reject changes to deal terms, with a clear record of what was changed and by whom. `sellerstatus` is redefined in v1.1 with a lifecycle-aligned enumeration; `buyerstatus` is added as its buyer-side counterpart. See [Deal Revision Workflow](implementation-guidance.md#deal-revision-workflow) for implementation guidance.
- **Unified Deal Endpoint:** The separate Status Endpoint from v1.0 is deprecated. The Deal API endpoint now supports both POST (for pushes and DealResponses) and GET (to retrieve the current state of a deal). GET responses return the Deal object directly, reflecting the responding party's current lifecycle state. The v1.0 `BuyerSeat` wrapper object is removed; per-seat status detail is now carried in a `seatstatuses` array on the Deal object, populated by the buyer's endpoint. The v1.0 `BuyerStatus` object is renamed to `SeatStatus`.
- **Idempotency and Conflict Resolution Guidance:** Version 1.1 adds explicit implementation guidance for idempotent message handling and deterministic conflict resolution. Push operations (Deal objects and DealResponses) are safe to retry using the `revisionid` as a natural idempotency key. Conflict priority rules define how to resolve concurrent updates: terminal states always dominate, stale revision responses are rejected with HTTP 409, and simultaneous revision proposals are resolved in the seller's favor. See [Idempotency](implementation-guidance.md#idempotency) and [Conflict Priority Rules](implementation-guidance.md#conflict-priority-rules) in the Implementation Guidance.

---

<a name="deal-api-specification"></a>
# Deal API Specification
The Deal API supports two operations: **POST** for pushing deal data (new deals, revisions, and DealResponses) and **GET** for retrieving the current state of a specific deal. At minimum, the buyer system must implement both operations — POST to receive deals and revisions pushed by the seller, and GET to allow the seller to query deal status. When both parties agree to support bidirectional communication, the seller system should also implement both operations. GET responses return the Deal object reflecting the responding party's current state. Configuration of endpoints — including URL paths, endpoint discovery, and authentication — is out of scope for this specification and is the responsibility of each implementing party.

<a name="object-deal"></a>
## Object: Deal

| Attribute | Type | Description |
|-----------|------|-------------|
| `id` | string; **required** | The canonical deal identifier used in OpenRTB bid requests. This is the ID that appears as `deal.id` in the bid stream and must match between buyer and seller systems for deal targeting to function. For seller-initiated deals this is assigned by the seller. For buyer-initiated deals, the buyer proposes a value but `id` is formally confirmed by the seller upon accepting the initial revision, since the seller (SSP) controls bid request construction. |
| `sellerdealid` | string | The deal's reference identifier in the seller's system namespace. For seller-initiated deals this will typically match `id`. Included to support cases where the seller's internal ID differs from the canonical bid-stream ID. |
| `buyerdealid` | string | The deal's reference identifier in the buyer's system namespace. Allows the buyer to maintain their own persistent link to the deal independent of the seller-assigned `id`. For buyer-initiated deals this should be populated by the buyer in the initial revision push. |
| `name` | string, recommended | Name of the deal as assigned by the initiating party. Note: This name may be displayed to the counterparty and should be chosen accordingly. |
| `created` | string | UTC timestamp in seconds in ISO-8601 of when the deal was first created. |
| `sellerstatus` | int | Lifecycle status of the deal from the seller's perspective:<br> `0` = pending — deal has been pushed, awaiting counterparty acceptance<br> `1` = not started — deal has been accepted but the flight start date has not yet been reached<br> `2` = live — deal is active and the seller is trafficking against it<br> `3` = live, not spending — deal is live but no spend has been observed within an expected window (seller diagnostic)<br> `4` = paused — seller has temporarily suspended the deal<br> `5` = completed — deal has reached its end date or delivery goal<br> `6` = expired — deal lapsed without being activated<br> `7` = canceled — deal was terminated prior to completion<br><br>The seller populates this field in all Deal pushes and status responses. See [Deal Revision Workflow](implementation-guidance.md#deal-revision-workflow) for transition rules. |
| `buyerstatus` | int | Lifecycle status of the deal from the buyer's perspective:<br> `0` = pending — deal received, awaiting buyer review or acceptance<br> `1` = not started — deal has been accepted but the flight start date has not yet been reached<br> `2` = live — buyer is actively trafficking against the deal<br> `3` = paused — buyer has temporarily suspended their use of the deal<br> `4` = completed — deal has reached its end date or delivery goal<br> `5` = expired — deal lapsed without being activated<br> `6` = canceled — deal was terminated prior to completion<br><br>The buyer populates this field in Deal pushes (bidirectional model) and status responses. In the baseline model the seller learns `buyerstatus` by polling the buyer's endpoint (GET). See [Deal Revision Workflow](implementation-guidance.md#deal-revision-workflow) for transition rules. |
| `currentrevision` | DealRevision object | The pending proposed revision, present only when a revision is in PROPOSED state (`negotiationstatus=0`). Absent when no revision is outstanding (i.e., when the deal is quiescent and `liverevision` fully describes the current terms). **Required on initial push** (before any revision has been accepted) since `liverevision` is not yet established. See [Object: DealRevision](#object-dealrevision) and [Deal Revision Workflow](implementation-guidance.md#deal-revision-workflow) for additional detail. |
| `liverevision` | DealRevision object | The last revision accepted by the counterparty (`negotiationstatus=1`). Absent if no revision has yet been accepted. When `currentrevision` is present, its delta fields represent changes relative to `liverevision`. See [Deal Revision Workflow](implementation-guidance.md#deal-revision-workflow) for additional detail. |
| `origin` | string, **required** | The advertising system domain of the business entity that will receive bid responses for the deal (typically the SSP running the auction). This field identifies the auction operator, not the party who initiated the deal. |
| `seller` | string, recommended | Canonical domain of the business entity who sold the deal. This may be the same as the origin or curator, but it also could be any intermediate seller. <br><br> [See Implementation Guidance for additional detail](implementation-guidance.md#origin-curator-and-seller) |
| `desc` | string | Short description for the deal to help the receiver locate the deal once it has been sent. It is strongly recommended to keep this field to 250 characters or less. |
| `wseat` | string array, recommended | Allowed list of buyer seats (e.g., advertisers, agencies) allowed to bid on this impression. <br><br>IDs of seats and knowledge of the buyer's customers to which they refer must be coordinated between bidders and the exchange <i>a priori</i>. <br><br>At most, only one of `wseat` and `bseat` should be used in the same request. Omission of both implies no seat restrictions. |
| `bseat` | string array | Block list of buyer seats (e.g., advertisers, agencies) restricted from bidding on this impression. IDs of seats and knowledge of the buyer's customers to which they refer must be coordinated between bidders and the exchange <i>a priori</i>. <br><br>At most, only one of `wseat` and `bseat` should be used in the same request. Omission of both implies no seat restrictions.|
| `adtypes` | int array | The format of the ad creative(s) supported by the inventory in the deal:<br> `1` = Banner<br> `2` = Video<br> `3` = Audio<br> `4` = Native<br><br>If this is empty or missing, the deal is assumed to apply to all types of ad creative. |
| `auxdata` | int | Indicates if there is non-publisher data (i.e. a non-publisher data layer) applied in this package with an associated fee where:<br> `0` = undisclosed<br> `1` = yes, at start of deal and will not be modified<br> `2` = yes and subject to change after start of deal<br> `3` = no and will not be modified<br> `4` = no and subject to change after start of deal<br><br>[See implementation guidance for additional detail](implementation-guidance.md#auxiliary-data) |
| `pubcount` | int | Indicates if there is more than one publishing company:<br> `0` = undisclosed<br> `1` = single publisher<br> `2` = multi-publisher<br>[See implementation guidance for additional detail](implementation-guidance.md#publisher-count) |
| `dinventory` | int | Indicates if the inventory for the deal is dynamic, meaning sites or applications included in the deal may update after the deal is live where:<br> `0` = undisclosed<br> `1` = inventory will NOT update once the deal goes live<br> `2` = inventory where this deal may run is updated dynamically. <br><br>[See implementation guidance for additional detail](implementation-guidance.md#dynamic-inventory) |
| `terms` | object | Terms of the deal, reflecting the current live state (i.e., the terms from `liverevision`). **Required when `liverevision` is present.** May be omitted on initial push when no revision has yet been accepted, in which case receivers should derive the deal terms from `currentrevision`. See [Object: Terms](#object-terms) for additional detail. |
| `inventory` | object | Information about the inventory included in the deal, reflecting the current live state. **Required when `liverevision` is present.** May be omitted on initial push when no revision has yet been accepted, in which case receivers should derive the inventory from `currentrevision`. For static inventory deals (`dinventory=1`), all five composition dimensions may be used. For dynamic inventory deals (`dinventory=2`), the non-site/app dimensions (`contentcomp`, `devicecomp`, `usercomp`) remain meaningful and are encouraged; `sitecomp` and `appcomp` may also be included but should generally carry `fidelity=1` unless the seller commits to keeping them current via the revision workflow. <br><br>See [Object: Inventory](#object-inventory) and [Relationship to dinventory](implementation-guidance.md#inventory-and-dinventory) for additional detail. |
| `curation` | object | Information about the curation package if applicable. <br><br>See [Object: Curation](#object-curation) for additional detail. |
| `seatstatuses` | SeatStatus object array | Per-seat operational status of the deal within the buyer's system. Populated by the buyer's endpoint on GET responses. Each entry describes the state of the deal for a specific buyer seat (e.g., pending approval, active, paused). The seller's endpoint does not populate this field. See [Object: SeatStatus](#object-seatstatus) for additional detail. |
| `ext` | object | Placeholder for deal-specific extensions |

<a name="object-terms"></a>
## Object: Terms

| Attribute | Type | Description |
|-----------|------|-------------|
| `startdate` | string | UTC timestamp in seconds in ISO-8601 of when the deal starts |
| `enddate` | string | UTC timestamp in seconds in ISO-8601 of when the deal ends. Evergreen or always on deals should leave this field blank. |
| `countries` | string array | An array of country codes in which the deal is available, where country code is a string using ISO-3166-3. If this is empty or missing, the deal is assumed to apply to all countries. |
| `dealfloor` | float | Minimum bid for impressions for this deal expressed in CPM. Unless `pricetype` is Fixed, this should be used as guidance to buyers. <br><br> [See Implementation Guidance for additional detail](implementation-guidance.md#price-and-floor-guidance) |
| `cur` | string; default "USD" | Bid currency using ISO-4217 alpha codes. |
| `guar` | int | Deal guarantee type where:<br> `0` = Not Guaranteed — the deal is biddable but the buyer is not obligated to bid<br> `1` = Guaranteed — the deal is guaranteed and the bidder must bid on the deal<br> `2` = Biddable Guaranteed — the deal is guaranteed with a commitment to deliver, but the buyer bids competitively rather than at a fixed price |
| `pricetype` | int, default 2 | Deal Price Type where:<br> `0` = Dynamic (ie. auction type will be provided by `request.at` attribute in OpenRTB Bid Request)<br> `1` = First Price<br> `2` = Second Price Plus<br> `3` = Fixed Price<br>Exchange-specific auction types can be defined using values 500 and greater. |
| `units` | int | Number of units (impressions) over the specified start and end date of the deal. If the deal is guaranteed, this number should be provided. If the deal is not guaranteed this may be omitted. |
| `totalcost` | float | The total cost over the specified start and end date of the deal. If the deal is guaranteed, this value should be provided. If the deal is not guaranteed this may be omitted. <br><br> [See Implementation Guidance for additional detail](implementation-guidance.md#price-and-floor-guidance) |
| `ext` | object | Placeholder for deal-specific extensions |

<a name="object-inventory"></a>
## Object: Inventory

The Inventory object describes the composition of inventory included in the deal using five optional composition sub-objects, each aligned to a corresponding OpenRTB 2.6 object. Each composition sub-object expresses the inventory profile through inclusion and exclusion arrays. Any combination of sub-objects may be present; omitted sub-objects imply no constraint along that dimension.

The fields specified within composition objects should reflect characteristics of the underlying supply that are expected to hold across all impression opportunities associated with the deal. Transient, per-impression signals — such as IP addresses, individual user identifiers, or other data that varies with each bid request — are not appropriate. See [Field Selection Guidance](implementation-guidance.md#field-selection-guidance) in the Implementation Guidance document for per-object recommendations and examples.

| Attribute | Type | Description |
|-----------|------|-------------|
| `contentcomp` | object | Content-based inventory composition. Describes the content context in which the deal's inventory will appear. See [Object: ContentComposition](#object-contentcomposition). |
| `devicecomp` | object | Device-based inventory composition. Describes the device types and characteristics of inventory included in the deal. See [Object: DeviceComposition](#object-devicecomposition). |
| `usercomp` | object | Curated audience composition. Describes the audience segments associated with inventory in the deal using the IAB Tech Lab Curated Audiences (`user.data`) signal structure. See [Object: UserComposition](#object-usercomposition). |
| `sitecomp` | object | Site-based inventory composition. Describes the web site inventory included in the deal. See [Object: SiteComposition](#object-sitecomposition). |
| `appcomp` | object | App-based inventory composition. Describes the mobile or CTV application inventory included in the deal. See [Object: AppComposition](#object-appcomposition). |
| `ext` | object | Placeholder for inventory-specific extensions |

---

<a name="object-contentcomposition"></a>
### Object: ContentComposition

Describes the content context of the inventory in terms of OpenRTB 2.6 Content objects. The `incl` array specifies content attributes that must be matched for inventory to qualify for the deal. The `excl` array specifies content attributes that disqualify inventory from the deal. Each entry in either array is a partial or fully specified OpenRTB 2.6 Content object; only the fields present in a given entry are considered when evaluating a match. See [Partial Object Matching](#partial-object-matching) and [Field Selection Guidance](#field-selection-guidance) for guidance.

| Attribute | Type | Description |
|-----------|------|-------------|
| `incl` | Content object array | Array of OpenRTB 2.6 Content objects representing content contexts to be included in the deal. An empty or absent array implies no content-based inclusion constraint. |
| `excl` | Content object array | Array of OpenRTB 2.6 Content objects representing content contexts to be excluded from the deal. An empty or absent array implies no content-based exclusion constraint. |
| `fidelity` | integer | Descriptive completeness of the `incl`/`excl` arrays: `0` = undisclosed, `1` = indicative, `2` = exhaustive. See [Fidelity](implementation-guidance.md#field-selection-guidance). |
| `ext` | object | Placeholder for composition-specific extensions |

---

<a name="object-devicecomposition"></a>
### Object: DeviceComposition

Describes the device profile of the inventory in terms of OpenRTB 2.6 Device objects. The `incl` array specifies device attributes that must be matched for inventory to qualify for the deal. The `excl` array specifies device attributes that disqualify inventory from the deal. Each entry in either array is a partial or fully specified OpenRTB 2.6 Device object; only the fields present in a given entry are considered when evaluating a match. See [Partial Object Matching](#partial-object-matching) and [Field Selection Guidance](#field-selection-guidance) for guidance.

| Attribute | Type | Description |
|-----------|------|-------------|
| `incl` | Device object array | Array of OpenRTB 2.6 Device objects representing device profiles to be included in the deal. An empty or absent array implies no device-based inclusion constraint. |
| `excl` | Device object array | Array of OpenRTB 2.6 Device objects representing device profiles to be excluded from the deal. An empty or absent array implies no device-based exclusion constraint. |
| `fidelity` | integer | Descriptive completeness of the `incl`/`excl` arrays: `0` = undisclosed, `1` = indicative, `2` = exhaustive. See [Fidelity](implementation-guidance.md#field-selection-guidance). |
| `ext` | object | Placeholder for composition-specific extensions |

---

<a name="object-usercomposition"></a>
### Object: UserComposition

Describes the curated audience composition of the deal in terms of OpenRTB 2.6 Data objects, as used within the `user.data` array per the IAB Tech Lab Curated Audiences standard (formerly Seller-Defined Audiences). Each Data object represents a single cohort provider and the audience segment IDs that provider assigns to the inventory's user base. The `segtax` extension on the Data object identifies the taxonomy — such as IAB Audience Taxonomy 1.x (segtax=4) or a vendor-specific taxonomy (segtax=500+) — within which the segment IDs are defined.

The `incl` array specifies audience data entries that must be matched for inventory to qualify for the deal. The `excl` array specifies audience data entries that disqualify inventory from the deal. Each entry is an OpenRTB 2.6 Data object. Only the fields that are present in a given entry are used as matching criteria; absent fields are treated as wildcards. A Data object with no `segment` array specified matches any impression carrying a signal from that provider under the given taxonomy. A Data object with a `segment` array matches only impressions where the user carries at least one of the specified segment IDs from that provider.

Note: Consistent with the Curated Audiences standard's design principles, UserComposition entries should convey anonymized taxonomy-based cohort signals and should not be commingled with device-specific identifiers, user-agent strings, or other data types that could pose privacy risks. Implementers should ensure compliance with applicable privacy regulations and consent frameworks.

| Attribute | Type | Description |
|-----------|------|-------------|
| `incl` | Data object array | Array of OpenRTB 2.6 Data objects (per `user.data` structure) representing curated audience segments to be included in the deal. Each Data object identifies a cohort provider via `name` (provider domain), specifies a taxonomy via `ext.segtax`, and optionally enumerates target segment IDs via `segment[].id`. An empty or absent array implies no audience-based inclusion constraint. |
| `excl` | Data object array | Array of OpenRTB 2.6 Data objects representing curated audience segments to be excluded from the deal. Structure mirrors that of `incl`. An empty or absent array implies no audience-based exclusion constraint. |
| `fidelity` | integer | Descriptive completeness of the `incl`/`excl` arrays: `0` = undisclosed, `1` = indicative, `2` = exhaustive. See [Fidelity](implementation-guidance.md#field-selection-guidance). |
| `ext` | object | Placeholder for composition-specific extensions |

---

<a name="object-sitecomposition"></a>
### Object: SiteComposition

Describes the web site inventory of the deal in terms of OpenRTB 2.6 Site objects. The `incl` array specifies site attributes that must be matched for inventory to qualify for the deal. The `excl` array specifies site attributes that disqualify inventory from the deal. Each entry in either array is a partial or fully specified OpenRTB 2.6 Site object; only the fields present in a given entry are considered when evaluating a match. See [Partial Object Matching](#partial-object-matching) and [Field Selection Guidance](#field-selection-guidance) for guidance.

It is strongly recommended that `incl` entries include the `publisher` object (with at minimum `publisher.domain`) and the `inventorypartnerdomain` field, to enable buyers to perform supply chain authorization checks in advance of the deal flight.

| Attribute | Type | Description |
|-----------|------|-------------|
| `incl` | Site object array | Array of OpenRTB 2.6 Site objects representing web site inventory to be included in the deal. An empty or absent array implies no site-based inclusion constraint. |
| `excl` | Site object array | Array of OpenRTB 2.6 Site objects representing web site inventory to be excluded from the deal. An empty or absent array implies no site-based exclusion constraint. |
| `fidelity` | integer | Descriptive completeness of the `incl`/`excl` arrays: `0` = undisclosed, `1` = indicative, `2` = exhaustive. See [Fidelity](implementation-guidance.md#field-selection-guidance). |
| `ext` | object | Placeholder for composition-specific extensions |

---

<a name="object-appcomposition"></a>
### Object: AppComposition

Describes the mobile or CTV application inventory of the deal in terms of OpenRTB 2.6 App objects. The `incl` array specifies app attributes that must be matched for inventory to qualify for the deal. The `excl` array specifies app attributes that disqualify inventory from the deal. Each entry in either array is a partial or fully specified OpenRTB 2.6 App object; only the fields present in a given entry are considered when evaluating a match. See [Partial Object Matching](#partial-object-matching) and [Field Selection Guidance](#field-selection-guidance) for guidance.

It is strongly recommended that `incl` entries include the `publisher` object (with at minimum `publisher.domain`) and the `inventorypartnerdomain` field, to enable buyers to perform supply chain authorization checks in advance of the deal flight.

| Attribute | Type | Description |
|-----------|------|-------------|
| `incl` | App object array | Array of OpenRTB 2.6 App objects representing application inventory to be included in the deal. An empty or absent array implies no app-based inclusion constraint. |
| `excl` | App object array | Array of OpenRTB 2.6 App objects representing application inventory to be excluded from the deal. An empty or absent array implies no app-based exclusion constraint. |
| `fidelity` | integer | Descriptive completeness of the `incl`/`excl` arrays: `0` = undisclosed, `1` = indicative, `2` = exhaustive. See [Fidelity](implementation-guidance.md#field-selection-guidance). |
| `ext` | object | Placeholder for composition-specific extensions |

---

<a name="object-curation"></a>
## Object: Curation

Information about the selection and organization of inventory using technology and data, with the goal of creating effective packages for advertisers through prepackaged or real-time operations.

| Attribute | Type | Description |
|-----------|------|-------------|
| `curator` | string | Canonical domain of the business entity that did the packaging of inventory, technology and/or data. Most often, this will be the seller of the deal. |
| `cdealid` | string | Unique identifier for the deal in the Curators namespace. |
| `curfeetype` | int | Fee type being applied for curation of the deal:<br> `0` = undisclosed<br> `1` = percentage of spend<br> `2` = flat fee<br> `3` = CPM<br> `4` = no fee is paid for curation services<br><br>[See Implementation Guidance for additional detail](implementation-guidance.md#curation-fee) |
| `ext` | object | Placeholder for deal-specific extensions |

<a name="object-dealrevision"></a>
## Object: DealRevision

A DealRevision records a proposed or accepted change to a deal's terms. Each revision carries the identity of the party who made it, the negotiation status, an optional comment, and a set of delta fields representing the term changes. See [Revision Semantics](implementation-guidance.md#revision-semantics) for the rules governing delta interpretation, pre-acceptance full-specification requirements, and the one-pending-at-a-time constraint.

| Attribute | Type | Description |
|-----------|------|-------------|
| `revisionid` | string; **required** | A UUID assigned by the initiating party that uniquely identifies this revision. Because each party generates UUIDs independently, revision IDs are guaranteed to be globally unique even when both parties propose a revision simultaneously. Parties should use `revisionid` — not position in a history array or `revisedate` — as the authoritative reference when accepting, rejecting, or superseding a specific revision. |
| `revisedate` | string; **required** | UTC timestamp in ISO-8601 of when this revision was created. |
| `revisedby` | DealActor object; **required** | The party who created this revision. See [Object: DealActor](#object-dealactor). |
| `negotiationstatus` | int; **required** | The negotiation status of this revision:<br> `0` = PROPOSED — revision has been submitted and is awaiting the counterparty's response<br> `1` = ACCEPTED — revision has been accepted by the counterparty and is now the live state of the deal<br> `2` = REJECTED — revision was rejected by the counterparty; `liverevision` (if any) remains the operative state<br> `3` = SUPERSEDED — revision was replaced by a newer revision before the counterparty could act on it |
| `comment` | string | Optional human-readable note from the revising party describing the reason for or nature of the change. |
| `name` | string | Delta: updated deal name. |
| `desc` | string | Delta: updated deal description. |
| `seller` | string | Delta: updated seller domain. |
| `wseat` | string array | Delta: updated allowed buyer seat list. Replaces the full array (not a partial merge). |
| `bseat` | string array | Delta: updated blocked buyer seat list. Replaces the full array (not a partial merge). |
| `adtypes` | int array | Delta: updated supported ad creative formats. Replaces the full array. |
| `auxdata` | int | Delta: updated auxiliary data indicator. |
| `pubcount` | int | Delta: updated publisher count indicator. |
| `dinventory` | int | Delta: updated dynamic inventory indicator. |
| `terms` | object | Delta: updated Terms object. Only fields that differ from `liverevision` need be included. See [Object: Terms](#object-terms). |
| `inventory` | object | Delta: updated Inventory object. Only fields that differ from `liverevision` need be included. See [Object: Inventory](#object-inventory). |
| `curation` | object | Delta: updated Curation object. Only fields that differ from `liverevision` need be included. See [Object: Curation](#object-curation). |
| `ext` | object | Placeholder for revision-specific extensions. |

The following Deal-level fields are **not revisionable** and cannot be changed via the revision workflow: `id`, `sellerdealid`, `buyerdealid`, `origin`, `created`, `sellerstatus`, `buyerstatus`, `currentrevision`, `liverevision`. These are structural or lifecycle fields managed by the protocol itself rather than by deal term negotiation.

---

<a name="object-dealactor"></a>
## Object: DealActor

Identifies the party who created a deal revision.

| Attribute | Type | Description |
|-----------|------|-------------|
| `partyid` | string; **required** | Identifier for the party in their own system's namespace (e.g., seat ID, account ID, or domain). |
| `contactemail` | string | Email address of the contact person at the revising party. |
| `role` | int; **required** | The role of the revising party:<br> `0` = SELLER — revision was created by the sell-side party (e.g., SSP or curator)<br> `1` = BUYER — revision was created by the buy-side party (e.g., DSP or agency) |
| `ext` | object | Placeholder for actor-specific extensions. |

<a name="object-dealresponse"></a>
## Object: DealResponse

A DealResponse communicates a party's acceptance or rejection of a proposed revision. Unlike a DealRevision (which proposes new terms), a DealResponse acts on an existing proposal without modifying the deal terms. DealResponses are pushed to the counterparty's push endpoint using an HTTP POST, the same endpoint used for Deal pushes. Implementations should distinguish between the two message types based on the payload structure: a DealResponse contains a top-level `revisionid` and `negotiationstatus` but no deal term fields.

| Attribute | Type | Description |
|-----------|------|-------------|
| `dealid` | string; **required** | The canonical deal identifier (`deal.id`) that this response pertains to. |
| `revisionid` | string; **required** | The UUID of the DealRevision being accepted or rejected. Must match the `currentrevision.revisionid` on the receiving party's system; if it does not, the response is stale and must be discarded (see [Stale acceptance and rejection](implementation-guidance.md#revision-semantics)). |
| `negotiationstatus` | int; **required** | The responding party's verdict on the referenced revision:<br> `1` = ACCEPTED — the counterparty accepts the proposed revision; it becomes `liverevision`<br> `2` = REJECTED — the counterparty rejects the proposed revision; `liverevision` (if any) remains the operative state |
| `respondedby` | DealActor object; **required** | The party issuing this response. See [Object: DealActor](#object-dealactor). |
| `responsedate` | string; **required** | UTC timestamp in ISO-8601 of when this response was issued. |
| `comment` | string | Optional human-readable note from the responding party explaining the acceptance or rejection. |
| `ext` | object | Placeholder for response-specific extensions. |

<a name="object-seatstatus"></a>
## Object: SeatStatus

Per-seat operational status of the deal within the buyer's system. Returned as entries in the `Deal.seatstatuses` array when the buyer's endpoint responds to a GET request. (This object was named `BuyerStatus` in v1.0 and was returned inside a `BuyerSeat` wrapper on the separate Status Endpoint. In v1.1, the Status Endpoint is deprecated and per-seat detail is carried directly on the Deal object.)

| Attribute | Type | Description |
|-----------|------|-------------|
| `buyerseatid` | string | Seat ID in the buying system that the response refers to |
| `status` | int | Operational status of the deal for this seat in the buying system:<br> `0` = pending approval<br> `1` = buyer has approved (ie. non PG deal is ready for bid requests)<br> `2` = buyer has rejected<br> `3` = ready to serve (ie. deal is in a campaign - ready to receive bid requests, relevant especially for PG deals)<br> `4` = active (i.e. deal is actively serving impressions)<br> `5` = paused<br> `6` = complete (ie. buying system shows deal has completed)|
| `ext` | object | Placeholder for seat-specific extensions |

The `seatstatuses` array includes only seats that have actively engaged with the deal. Seats that have not interacted with the deal are simply absent from the array. For open deals where both `wseat` and `bseat` are omitted on the Deal object (e.g., library or evergreen deals with no per-seat approval gate), seats may begin at `status=1` (approved) or `status=3` (ready to serve) rather than `status=0` (pending approval), since no approval step is required.

---

<a name="implementation-guidance"></a>
# Implementation Guidance

Implementation guidance for the Deal Sync API v1.1 — including matching bid requests to deals, authorization, sending and receiving information, idempotency, conflict resolution, deal object guidance, inventory composition, deal revision workflow, pricing, and curation — is provided in the companion document [implementation-guidance.md](implementation-guidance.md).
