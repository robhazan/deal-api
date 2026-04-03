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
- [Status Endpoint](#receiver-endpoint)
  - [Object: BuyerSeat](#object-buyerseat)
  - [Object: SeatStatus](#object-seatstatus)
- [Implementation Guidance](#implementation-guidance)
  - [Matching Bid Requests to Deals](#matching-bid-requests-to-deals)
  - [Authorization](#authorization)
  - [Sending and Receiving Information](#sending-and-receiving-information)
  - [Deal Object Guidance](#deal-object-guidance)
    - [Auxiliary Data](#auxiliary-data)
    - [Publisher Count](#publisher-count)
    - [Dynamic Inventory](#dynamic-inventory)
  - [Terms Object](#terms-object-1)
  - [Inventory Object](#inventory-object-1)
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
  - [Curation Object](#curation-object-1)
    - [Curation Fee](#curation-fee)
  - [Example Scenarios](#example-scenarios)

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
- **Deal Revision Workflow:** Version 1.1 introduces a structured revision lifecycle for deal terms, replacing the need for ad-hoc differential overrides. All updates to deal terms — whether minor adjustments or substantive renegotiations — flow through the same revision workflow: a party proposes a revision, and the counterparty accepts or rejects it. The `Deal` object gains `currentrevision`, `liverevision`, `sellerstatus`, and `buyerstatus` fields. The new `DealRevision`, `DealActor`, and `DealResponse` objects provide a standardized mechanism for either party to propose, accept, or reject changes to deal terms, with a clear record of what was changed and by whom. `sellerstatus` is redefined in v1.1 with a lifecycle-aligned enumeration; `buyerstatus` is added as its buyer-side counterpart. See [Deal Revision Workflow](#deal-revision-workflow) for implementation guidance.

---

<a name="deal-api-specification"></a>
# Deal API Specification
An HTTP POST endpoint for accepting deal data. At minimum, the buyer system must implement this endpoint to receive deals and revisions pushed by the seller (the traditional seller-push model). When both parties agree to support bidirectional communication, the seller system should also implement this endpoint to receive buyer-initiated revisions and DealResponses. Configuration of push calls — including endpoint discovery and authentication — is out of scope for this specification and is the responsibility of each implementing party.

<a name="object-deal"></a>
## Object: Deal

| Attribute | Type | Description |
|-----------|------|-------------|
| `id` | string; **required** | The canonical deal identifier used in OpenRTB bid requests. This is the ID that appears as `deal.id` in the bid stream and must match between buyer and seller systems for deal targeting to function. For seller-initiated deals this is assigned by the seller. For buyer-initiated deals, the buyer proposes a value but `id` is formally confirmed by the seller upon accepting the initial revision, since the seller (SSP) controls bid request construction. |
| `sellerdealid` | string | The deal's reference identifier in the seller's system namespace. For seller-initiated deals this will typically match `id`. Included to support cases where the seller's internal ID differs from the canonical bid-stream ID. |
| `buyerdealid` | string | The deal's reference identifier in the buyer's system namespace. Allows the buyer to maintain their own persistent link to the deal independent of the seller-assigned `id`. For buyer-initiated deals this should be populated by the buyer in the initial revision push. |
| `name` | string, recommended | Name of the deal as assigned by the initiating party. Note: This name may be displayed to the counterparty and should be chosen accordingly. |
| `created` | string | UTC timestamp in seconds in ISO-8601 of when the deal was first created. |
| `sellerstatus` | int | Lifecycle status of the deal from the seller's perspective:<br> `0` = pending — deal has been pushed, awaiting counterparty acceptance<br> `1` = not started — deal has been accepted but the flight start date has not yet been reached<br> `2` = live — deal is active and the seller is trafficking against it<br> `3` = live, not spending — deal is live but no spend has been observed within an expected window (seller diagnostic)<br> `4` = paused — seller has temporarily suspended the deal<br> `5` = completed — deal has reached its end date or delivery goal<br> `6` = expired — deal lapsed without being activated<br> `7` = canceled — deal was terminated prior to completion<br><br>The seller populates this field in all Deal pushes and status responses. See [Deal Revision Workflow](#deal-revision-workflow) for transition rules. |
| `buyerstatus` | int | Lifecycle status of the deal from the buyer's perspective:<br> `0` = pending — deal received, awaiting buyer review or acceptance<br> `1` = not started — deal has been accepted but the flight start date has not yet been reached<br> `2` = live — buyer is actively trafficking against the deal<br> `3` = paused — buyer has temporarily suspended their use of the deal<br> `4` = completed — deal has reached its end date or delivery goal<br> `5` = expired — deal lapsed without being activated<br> `6` = canceled — deal was terminated prior to completion<br><br>The buyer populates this field in Deal pushes (bidirectional model) and status responses. In the baseline model the seller learns `buyerstatus` by polling the buyer's status endpoint. See [Deal Revision Workflow](#deal-revision-workflow) for transition rules. |
| `currentrevision` | DealRevision object | The pending proposed revision, present only when a revision is in PROPOSED state (`negotiationstatus=0`). Absent when no revision is outstanding (i.e., when the deal is quiescent and `liverevision` fully describes the current terms). **Required on initial push** (before any revision has been accepted) since `liverevision` is not yet established. See [Object: DealRevision](#object-dealrevision) and [Deal Revision Workflow](#deal-revision-workflow) for additional detail. |
| `liverevision` | DealRevision object | The last revision accepted by the counterparty (`negotiationstatus=1`). Absent if no revision has yet been accepted. When `currentrevision` is present, its delta fields represent changes relative to `liverevision`. See [Deal Revision Workflow](#deal-revision-workflow) for additional detail. |
| `origin` | string, **required** | The advertising system domain of the business entity that will receive bid responses for the deal (typically the SSP running the auction). This field identifies the auction operator, not the party who initiated the deal. |
| `seller` | string, recommended | Canonical domain of the business entity who sold the deal. This may be the same as the origin or curator, but it also could be any intermediate seller. <br><br> [See Implementation Guidance for additional detail](#origin-curator-and-seller) |
| `desc` | string | Short description for the deal to help the receiver locate the deal once it has been sent. It is strongly recommended to keep this field to 250 characters or less. |
| `wseat` | string array, recommended | Allowed list of buyer seats (e.g., advertisers, agencies) allowed to bid on this impression. <br><br>IDs of seats and knowledge of the buyer's customers to which they refer must be coordinated between bidders and the exchange <i>a priori</i>. <br><br>At most, only one of `wseat` and `bseat` should be used in the same request. Omission of both implies no seat restrictions. |
| `bseat` | string array | Block list of buyer seats (e.g., advertisers, agencies) restricted from bidding on this impression. IDs of seats and knowledge of the buyer's customers to which they refer must be coordinated between bidders and the exchange <i>a priori</i>. <br><br>At most, only one of `wseat` and `bseat` should be used in the same request. Omission of both implies no seat restrictions.|
| `adtypes` | int array | The format of the ad creative(s) supported by the inventory in the deal:<br> `1` = Banner<br> `2` = Video<br> `3` = Audio<br> `4` = Native<br><br>If this is empty or missing, the deal is assumed to apply to all types of ad creative. |
| `auxdata` | int | Indicates if there is non-publisher data (i.e. a non-publisher data layer) applied in this package with an associated fee where:<br> `0` = undisclosed<br> `1` = yes, at start of deal and will not be modified<br> `2` = yes and subject to change after start of deal<br> `3` = no and will not be modified<br> `4` = no and subject to change after start of deal<br><br>[See implementation guidance for additional detail](#auxiliary-data) |
| `pubcount` | int | Indicates if there is more than one publishing company:<br> `0` = undisclosed<br> `1` = single publisher<br> `2` = multi-publisher<br>[See implementation guidance for additional detail](#publisher-count) |
| `dinventory` | int | Indicates if the inventory for the deal is dynamic, meaning sites or applications included in the deal may update after the deal is live where:<br> `0` = undisclosed<br> `1` = inventory will NOT update once the deal goes live<br> `2` = inventory where this deal may run is updated dynamically. <br><br>[See implementation guidance for additional detail](#dynamic-inventory) |
| `terms` | object | Terms of the deal, reflecting the current live state (i.e., the terms from `liverevision`). **Required when `liverevision` is present.** May be omitted on initial push when no revision has yet been accepted, in which case receivers should derive the deal terms from `currentrevision`. See [Object: Terms](#object-terms) for additional detail. |
| `inventory` | object | Information about the inventory included in the deal, reflecting the current live state. **Required when `liverevision` is present.** May be omitted on initial push when no revision has yet been accepted, in which case receivers should derive the inventory from `currentrevision`. For static inventory deals (`dinventory=1`), all five composition dimensions may be used. For dynamic inventory deals (`dinventory=2`), the non-site/app dimensions (`contentcomp`, `devicecomp`, `usercomp`) remain meaningful and are encouraged; `sitecomp` and `appcomp` may also be included but should generally carry `fidelity=1` unless the seller commits to keeping them current via the revision workflow. <br><br>See [Object: Inventory](#object-inventory) and [Relationship to dinventory](#inventory-and-dinventory) for additional detail. |
| `curation` | object | Information about the curation package if applicable. <br><br>See [Object: Curation](#object-curation) for additional detail. |
| `ext` | object | Placeholder for deal-specific extensions |

<a name="object-terms"></a>
## Object: Terms

| Attribute | Type | Description |
|-----------|------|-------------|
| `startdate` | string | UTC timestamp in seconds in ISO-8601 of when the deal starts |
| `enddate` | string | UTC timestamp in seconds in ISO-8601 of when the deal ends. Evergreen or always on deals should leave this field blank. |
| `countries` | string array | An array of country codes in which the deal is available, where country code is a string using ISO-3166-3. If this is empty or missing, the deal is assumed to apply to all countries. |
| `dealfloor` | float | Minimum bid for impressions for this deal expressed in CPM. Unless `pricetype` is Fixed, this should be used as guidance to buyers. <br><br> [See Implementation Guidance for additional detail](#price-and-floor-guidance) |
| `cur` | string; default "USD" | Bid currency using ISO-4217 alpha codes. |
| `guar` | int | Indicates that the deal is of type guaranteed and the bidder must bid on the deal, where 0 = not a guaranteed deal, 1 = guaranteed deal. |
| `pricetype` | int, default 2 | Deal Price Type where:<br> `0` = Dynamic (ie. auction type will be provided by `request.at` attribute in OpenRTB Bid Request)<br> `1` = First Price<br> `2` = Second Price Plus<br> `3` = Fixed Price<br>Exchange-specific auction types can be defined using values 500 and greater. |
| `units` | int | Number of units (impressions) over the specified start and end date of the deal. If the deal is guaranteed, this number should be provided. If the deal is not guaranteed this may be omitted. |
| `totalcost` | float | The total cost over the specified start and end date of the deal. If the deal is guaranteed, this value should be provided. If the deal is not guaranteed this may be omitted. <br><br> [See Implementation Guidance for additional detail](#price-and-floor-guidance) |
| `ext` | object | Placeholder for deal-specific extensions |

<a name="object-inventory"></a>
## Object: Inventory

The Inventory object describes the composition of inventory included in the deal using five optional composition sub-objects, each aligned to a corresponding OpenRTB 2.6 object. Each composition sub-object expresses the inventory profile through inclusion and exclusion arrays. Any combination of sub-objects may be present; omitted sub-objects imply no constraint along that dimension.

The fields specified within composition objects should reflect characteristics of the underlying supply that are expected to hold across all impression opportunities associated with the deal. Transient, per-impression signals — such as IP addresses, individual user identifiers, or other data that varies with each bid request — are not appropriate. See [Field Selection Guidance](#field-selection-guidance) in the Implementation Guidance section for per-object recommendations and examples.

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
| `fidelity` | integer | Descriptive completeness of the `incl`/`excl` arrays: `0` = undisclosed, `1` = indicative, `2` = exhaustive. See [Fidelity](#field-selection-guidance). |
| `ext` | object | Placeholder for composition-specific extensions |

---

<a name="object-devicecomposition"></a>
### Object: DeviceComposition

Describes the device profile of the inventory in terms of OpenRTB 2.6 Device objects. The `incl` array specifies device attributes that must be matched for inventory to qualify for the deal. The `excl` array specifies device attributes that disqualify inventory from the deal. Each entry in either array is a partial or fully specified OpenRTB 2.6 Device object; only the fields present in a given entry are considered when evaluating a match. See [Partial Object Matching](#partial-object-matching) and [Field Selection Guidance](#field-selection-guidance) for guidance.

| Attribute | Type | Description |
|-----------|------|-------------|
| `incl` | Device object array | Array of OpenRTB 2.6 Device objects representing device profiles to be included in the deal. An empty or absent array implies no device-based inclusion constraint. |
| `excl` | Device object array | Array of OpenRTB 2.6 Device objects representing device profiles to be excluded from the deal. An empty or absent array implies no device-based exclusion constraint. |
| `fidelity` | integer | Descriptive completeness of the `incl`/`excl` arrays: `0` = undisclosed, `1` = indicative, `2` = exhaustive. See [Fidelity](#field-selection-guidance). |
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
| `fidelity` | integer | Descriptive completeness of the `incl`/`excl` arrays: `0` = undisclosed, `1` = indicative, `2` = exhaustive. See [Fidelity](#field-selection-guidance). |
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
| `fidelity` | integer | Descriptive completeness of the `incl`/`excl` arrays: `0` = undisclosed, `1` = indicative, `2` = exhaustive. See [Fidelity](#field-selection-guidance). |
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
| `fidelity` | integer | Descriptive completeness of the `incl`/`excl` arrays: `0` = undisclosed, `1` = indicative, `2` = exhaustive. See [Fidelity](#field-selection-guidance). |
| `ext` | object | Placeholder for composition-specific extensions |

---

<a name="object-curation"></a>
## Object: Curation

Information about the selection and organization of inventory using technology and data, with the goal of creating effective packages for advertisers through prepackaged or real-time operations.

| Attribute | Type | Description |
|-----------|------|-------------|
| `curator` | string | Canonical domain of the business entity that did the packaging of inventory, technology and/or data. Most often, this will be the seller of the deal. |
| `cdealid` | string | Unique identifier for the deal in the Curators namespace. |
| `curfeetype` | int | Fee type being applied for curation of the deal:<br> `0` = undisclosed<br> `1` = percentage of spend<br> `2` = flat fee<br> `3` = CPM<br> `4` = no fee is paid for curation services<br><br>[See Implementation Guidance for additional detail](#curation-fee) |
| `ext` | object | Placeholder for deal-specific extensions |

<a name="object-dealrevision"></a>
## Object: DealRevision

A DealRevision records a proposed or accepted change to a deal's terms. Each revision carries the identity of the party who made it, the negotiation status, an optional comment, and a set of delta fields representing the term changes. See [Revision Semantics](#revision-semantics) for the rules governing delta interpretation, pre-acceptance full-specification requirements, and the one-pending-at-a-time constraint.

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
| `revisionid` | string; **required** | The UUID of the DealRevision being accepted or rejected. Must match the `currentrevision.revisionid` on the receiving party's system; if it does not, the response is stale and must be discarded (see [Stale acceptance and rejection](#revision-semantics)). |
| `negotiationstatus` | int; **required** | The responding party's verdict on the referenced revision:<br> `1` = ACCEPTED — the counterparty accepts the proposed revision; it becomes `liverevision`<br> `2` = REJECTED — the counterparty rejects the proposed revision; `liverevision` (if any) remains the operative state |
| `respondedby` | DealActor object; **required** | The party issuing this response. See [Object: DealActor](#object-dealactor). |
| `responsedate` | string; **required** | UTC timestamp in ISO-8601 of when this response was issued. |
| `comment` | string | Optional human-readable note from the responding party explaining the acceptance or rejection. |
| `ext` | object | Placeholder for response-specific extensions. |

---

<a name="receiver-endpoint"></a>
# Status Endpoint
An HTTP GET endpoint for requesting current information about a specific deal. At minimum, the buyer system must implement this endpoint so the seller can query deal status. When both parties agree to support bidirectional communication, the seller system should also implement this endpoint to allow the buyer to query deal state.

<a name="object-buyerseat"></a>
## Object: BuyerSeat

Information about the status of the deal in the buying system at a seat level.

| Attribute | Type | Description |
|-----------|------|-------------|
| `version` | string, **required** | Version of the Deal API in use |
| `id` | string, **required** | A unique identifier for the deal as passed in the initial push that this response is referring to. This should always be the same id as `deal.id` |
| `seatstatuses` | object array | Information about the buying seat where the Deal will be trafficked. (Renamed from `buyerstatus` in v1.1 to avoid collision with the Deal-level `buyerstatus` lifecycle field.) |
| `ext` | object | Placeholder for deal-specific extensions |

<a name="object-seatstatus"></a>
## Object: SeatStatus

Information about the status of the deal at a seat level in the buying system. (Renamed from `BuyerStatus` in v1.1.)

| Attribute | Type | Description |
|-----------|------|-------------|
| `buyerseatid` | string | Seat ID in the buying system that the response refers to |
| `status` | int | Status of the deal in the buying system:<br> `0` = pending approval<br> `1` = buyer has approved (ie. non PG deal is ready for bid requests)<br> `2` = buyer has rejected<br> `3` = ready to serve (ie. deal is in a campaign - ready to receive bid requests, relevant especially for PG deals)<br> `4` = active (i.e. deal is actively serving impressions)<br> `5` = paused<br> `6` = complete (ie. buying system shows deal has completed)|
| `ext` | object | Placeholder for deal-specific extensions |

---

<a name="implementation-guidance"></a>
# Implementation Guidance

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

**Baseline (seller-push) model:** At minimum, the seller initiates deals and proposes revisions by pushing Deal objects to the buyer's push endpoint, and periodically polls the buyer's status endpoint to retrieve the current state of the deal. The buyer must implement the push endpoint (to receive incoming deals and revisions) and the status endpoint (to respond to queries). In this model, the buyer communicates acceptance or rejection by updating the `negotiationstatus` on `currentrevision` in the Deal object returned via the status endpoint. The seller detects the buyer's verdict on its next poll. This model does not require the seller to implement any endpoint.

**Bidirectional model (optional):** When both parties agree to support bidirectionality, either party may initiate a deal or propose a revision by pushing to the counterparty's push endpoint, and both parties implement both the push and status endpoints. In this model, the buyer can also push DealResponse objects directly to the seller's push endpoint for faster acceptance/rejection notification, and may initiate deals or propose revisions of their own. The conflict resolution rules described in [Revision Semantics](#revision-semantics) (simultaneous proposals, stale acceptance) apply only when both parties are actively pushing.

**Push endpoint message types:** The push endpoint accepts two message types via HTTP POST:

- **Deal object** — used to create a new deal or propose a revision. The Deal object contains `currentrevision` with the proposed changes.
- **DealResponse object** — used to accept or reject an existing proposed revision. The DealResponse references the target revision by `revisionid` and communicates the verdict via `negotiationstatus`.

Implementations should distinguish between the two based on the payload structure: a DealResponse has a top-level `revisionid` and `negotiationstatus` with no deal term fields; a Deal push has the full Deal object structure. A counter-revision (proposing alternative terms in response to a proposal) is communicated as a new Deal push, not as a DealResponse.

To ensure data integrity, implementers are encouraged to implement periodic polling as a fallback mechanism to handle any missed push notifications.

Implementers may choose to accept incoming webhooks to their API endpoints for events. Please discuss this feature and support with your chosen integration partners.

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

<a name="terms-object-1"></a>
## Terms Object

Price attribute must be provided if the deal is a fixed price deal, especially if it's a programmatic guaranteed deal, but otherwise can be used as pricing guidance or not provided at all if there is no pricing guidance (for example, for a Run-of-Exchange deal where there is no pricing guidance).

Where multiple DSP seats are included, per seat acceptance/rejection is on the DSP to surface in their internal systems and User Interfaces.

<a name="inventory-object-1"></a>
## Inventory Object

For static inventory deals (`dinventory=1`), all five composition dimensions may be used. For dynamic inventory deals (`dinventory=2`), the non-site/app composition dimensions (`contentcomp`, `devicecomp`, `usercomp`) remain meaningful and are encouraged, as they describe the profile of the supply rather than enumerating specific properties. `sitecomp` and `appcomp` may also be included for dynamic deals — for example, to identify publishers and support advance supply authorization — but should generally carry `fidelity=1` unless the seller is prepared to keep them current via the revision workflow. See [Relationship to dinventory](#inventory-and-dinventory) for the full interaction guidance.

<a name="composition-object-design"></a>
### Composition Object Design

The v1.1 Inventory object replaces the flat attribute model from v1.0 with a composition-based model aligned to OpenRTB 2.6. Five sub-objects — Content, Device, Audience, Site, and App — each use `incl`/`excl` arrays of their corresponding OpenRTB 2.6 objects to express inventory profiles at any level of granularity. The exception is `usercomp`, which uses OpenRTB 2.6 **Data** objects (the `user.data` sub-structure) rather than full User objects, consistent with the IAB Tech Lab Curated Audiences standard. See [Object: UserComposition](#object-usercomposition) for details and refer to the Curated Audiences specification for `segtax` taxonomy enumeration.

Each composition sub-object also carries a `fidelity` field — see [Field Selection Guidance](#field-selection-guidance) for definition and usage.

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

Composition objects are intended to characterize the nature of the supply across the deal as a whole — not to enumerate every possible value a field might take on a given impression. When deciding which fields to populate in a composition entry, the guiding question is: "Is this characteristic expected to hold across all (or nearly all) impression opportunities in this deal?" If so, it is a good candidate. If the value is transient, per-impression, or user-session specific, it is not appropriate here.

The following guidance applies to each composition sub-object.

**Fidelity**

Each composition sub-object carries a `fidelity` field that communicates how comprehensively its `incl` and `excl` arrays describe the deal's supply along that dimension:

- `fidelity = 1` (indicative): The composition characterizes the general shape of the supply but should not be taken as a complete picture. Some impressions delivered on the deal may not match all specified dimensions. This is appropriate when the seller can describe the dominant characteristics of the supply but cannot enumerate every variation that may appear — for example, a content profile describing the prevailing genre and language of a broadly curated package.
- `fidelity = 2` (exhaustive): The composition comprehensively describes the supply. Buyers should not expect impressions that fall outside the specified dimensions. This is appropriate when the seller can enumerate the full set of properties, applications, or audience segments covered by the deal — for example, a fixed list of owned-and-operated sites.

Because `fidelity` is declared per composition sub-object, different dimensions within the same deal may carry different declarations. A deal might warrant `sitecomp.fidelity = 2` (the site list is complete and will not change) while setting `contentcomp.fidelity = 1` (the content profile is representative but not exhaustive). Buyers should evaluate each dimension's fidelity independently when deciding how to apply the composition signals for targeting and validation.

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

The `dinventory` field on the Deal object and the `fidelity` field on each composition sub-object describe different aspects of the deal's supply and are complementary rather than redundant.

`dinventory` is a forward-looking statement about **temporal stability**: will the set of sites and applications included in this deal change after it goes live? `fidelity` is a statement about **descriptive completeness**: how faithfully do the composition objects describe the supply as of the time the deal was sent?

Because they operate on different axes, each of the four meaningful combinations carries distinct implications:

- **`dinventory=1` + `fidelity=2`** (static supply, exhaustively described): The composition is a complete picture of the supply and that supply will not change. This is the strongest signal a seller can provide and gives buyers the highest confidence for advance targeting and validation.
- **`dinventory=1` + `fidelity=1`** (static supply, described indicatively): The supply will not change, but the composition only approximates it. This is valid, though sellers are encouraged to upgrade to `fidelity=2` where possible — if the supply is static, a complete enumeration is in principle achievable.
- **`dinventory=2` + `fidelity=1`** (dynamic supply, described indicatively): The supply evolves over the flight of the deal and the composition describes its general profile. This is the most common pairing for broadly curated or data-driven packages.
- **`dinventory=2` + `fidelity=2`** (dynamic supply, exhaustively described as of send time): The composition is a complete picture of the supply *at the time the deal was sent*. Because `dinventory=2` signals that inventory may change, sellers using this combination are obligated to push composition updates via the revision workflow whenever the supply changes materially — otherwise the `fidelity=2` declaration becomes misleading. Buyers should treat a `fidelity=2` composition on a dynamic deal as reliable only until a subsequent update arrives or until the deal flight warrants re-validation.

**Including the Inventory object for dynamic deals**

Prior to v1.1, the Inventory object was restricted to `dinventory=1` because it consisted of specific site and app lists that only made sense for fixed supply. The v1.1 composition model changes this. Dimensions such as `contentcomp`, `devicecomp`, and `usercomp` describe the *nature and profile* of the supply rather than enumerating specific properties — and that profile is meaningful and useful regardless of whether the supply is static or dynamic. Sellers are encouraged to include these dimensions for dynamic deals to give buyers useful advance signal about the content environments, device types, and audience segments they should expect.

For `sitecomp` and `appcomp` in a dynamic deal, the composition can still be valuable — for example, to identify the publisher and support supply authorization checks — but `fidelity=1` (indicative) is appropriate unless the seller is prepared to maintain an exhaustive and current list via the revision workflow as inventory changes.

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

**Stale acceptance and rejection:** Any DealResponse pushed by a party must reference the `revisionid` of the revision being acted upon. Upon receiving a DealResponse, the receiving party must validate that the referenced `revisionid` matches their current `currentrevision.revisionid`. If the IDs do not match — because the revision has since been superseded — the DealResponse is stale and must be discarded. The responding party should re-evaluate `currentrevision` and respond to the correct revision. See [Object: DealResponse](#object-dealresponse) for the message format.

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

By default, the Deal API returns only `currentrevision` and `liverevision` on the Deal object. When the counterparty's status endpoint supports it, either party may append `full_history=1` to the query parameters to request the complete ordered array of all prior revisions for the deal. This allows either party to audit the full negotiation history — including all PROPOSED, REJECTED, and SUPERSEDED revisions — when needed for dispute resolution or deal analysis. Implementations are not required to retain or serve full revision history, but are encouraged to do so.

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

<a name="curation-object-1"></a>
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
