# Deal Sync API v1.1 — Example Workflows

## Table of Contents

- [Scenario 1: Saleshouse Library Deal Across Multiple Web Publishers](#scenario-1-saleshouse-library-deal-across-multiple-web-publishers)
  - [Step 1: Saleshouse Creates the Deal](#step-1-saleshouse-creates-the-deal)
  - [Step 2: Seller Polls Buyer's Endpoint (Accepted, Not Yet Live)](#step-2-seller-polls-buyers-endpoint-accepted-not-yet-live)
  - [Step 3: Seller Polls Buyer's Endpoint (Live, Multiple Seats Active)](#step-3-seller-polls-buyers-endpoint-live-multiple-seats-active)
- [Scenario 2: Seller-Initiated CTV Deal with Buyer Revision](#scenario-2-seller-initiated-ctv-deal-with-buyer-revision)
  - [Step 1: Seller Creates the Deal](#step-1-seller-creates-the-deal)
  - [Step 2: Buyer Accepts the Initial Proposal](#step-2-buyer-accepts-the-initial-proposal)
  - [Step 3: Buyer Proposes a Revision](#step-3-buyer-proposes-a-revision)
  - [Step 4: Seller Accepts the Revision](#step-4-seller-accepts-the-revision)
  - [Step 5: Buyer Polls Seller's Endpoint](#step-5-buyer-polls-sellers-endpoint)
- [Scenario 3: Buyer-Initiated Mobile Gaming Deal with Seller Revision](#scenario-3-buyer-initiated-mobile-gaming-deal-with-seller-revision)
  - [Step 1: Buyer Creates the Deal](#step-1-buyer-creates-the-deal)
  - [Step 2: Seller Accepts the Initial Proposal](#step-2-seller-accepts-the-initial-proposal-1)
  - [Step 3: Seller Proposes a Revision (Console Expansion)](#step-3-seller-proposes-a-revision-console-expansion)
  - [Step 4: Buyer Accepts the Revision](#step-4-buyer-accepts-the-revision-1)
  - [Step 5: Seller Pauses the Deal](#step-5-seller-pauses-the-deal)
  - [Step 6: Buyer Cancels the Deal](#step-6-buyer-cancels-the-deal)

---

<a name="scenario-1-saleshouse-library-deal-across-multiple-web-publishers"></a>
## Scenario 1: Saleshouse Library Deal Across Multiple Web Publishers

A technical saleshouse (PremiumWeb Group, `premiumwebgroup.com`) represents three cooking and recipe web publishers and creates a **library deal** (also known as an evergreen deal) packaging their combined inventory against an in-market auto intender audience defined using IAB Audience Taxonomy 1.1 signals. The deal covers banner and outstream video creatives. Because the saleshouse may adjust its publisher roster over the flight, inventory is dynamic (`dinventory=2`). The saleshouse charges a CPM curation fee for the audience service.

As a library deal, there is no end date and no seat restrictions — both `wseat` and `bseat` are omitted, signaling that any buyer seat may target this deal without prior approval. Individual seats appear in `seatstatuses` as they begin engaging with the deal, skipping the `status=0` (PENDING) approval gate.

This scenario uses the **baseline seller-push model**: the saleshouse pushes the deal to the buyer's endpoint, then polls the buyer's endpoint (GET) to learn when the buyer has accepted. The buyer does not implement a push endpoint — they communicate acceptance by updating the deal state on their side, which the seller discovers on its next poll. A second poll after the deal goes live demonstrates the multi-seat pattern typical of library deals.

---

### Step 1: Saleshouse Creates the Deal

The saleshouse pushes the initial library deal to the buyer's push endpoint. The audience composition (`usercomp`) is the primary targeting signal — content and device composition are included but play a supporting role (brand safety and format eligibility respectively). Because no revision has yet been accepted, `currentrevision` carries the full deal specification.

`sitecomp` is included to identify the current publisher roster and support supply chain authorization, with the understanding that the list may change over the flight. Note that `enddate` is omitted (evergreen), and neither `wseat` nor `bseat` is present — any buyer seat may target this deal.

**Request**
```
POST https://dsp.buyerco.com/deal-sync/v1/push
Content-Type: application/json
```

**Payload**
```json
{
  "id": "deal-web-auto-q2-001",
  "sellerdealid": "PWG-2026-AUTO-887",
  "name": "In-Market Auto Intenders — Web (Library Deal)",
  "desc": "Evergreen multi-publisher web deal targeting in-market auto intenders via IAB Audience Taxonomy signals, served across cooking and recipe publisher inventory. Banner and outstream video. Publisher roster subject to change. Open to all buyer seats.",
  "origin": "adxchange.io",
  "seller": "premiumwebgroup.com",
  "created": "2026-03-24T08:00:00Z",
  "sellerstatus": 0,
  "curation": {
    "curator": "premiumwebgroup.com",
    "cdealid": "PWG-CUR-2026-AUTO-887",
    "curfeetype": 3
  },
  "currentrevision": {
    "revisionid": "f2e9c4b1-7a3d-4f8e-b6c2-9d5a1e7f4b08",
    "revisedate": "2026-03-24T08:00:00Z",
    "revisedby": {
      "partyid": "premiumwebgroup.com",
      "contactemail": "programmatic@premiumwebgroup.com",
      "role": 0
    },
    "negotiationstatus": 0,
    "comment": "Initial proposal. Evergreen library deal — audience defined via IAB Audience Taxonomy 1.1 in-market auto intender segments, overlaid on cooking and recipe publisher inventory. Publisher list subject to change; content and device dimensions are indicative. Open to all buyer seats.",
    "adtypes": [1, 2],
    "auxdata": 2,
    "pubcount": 2,
    "dinventory": 2,
    "terms": {
      "startdate": "2026-04-01T00:00:00Z",
      "countries": ["USA"],
      "dealfloor": 4.50,
      "cur": "USD",
      "pricetype": 2,
      "guar": 0
    },
    "inventory": {
      "usercomp": {
        "incl": [
          {
            "name": "premiumwebgroup.com",
            "ext": { "segtax": 4 },
            "segment": [
              { "id": "284" },
              { "id": "285" },
              { "id": "287" }
            ]
          },
          {
            "name": "datapartner.com",
            "ext": { "segtax": 4 },
            "segment": [
              { "id": "284" },
              { "id": "286" }
            ]
          }
        ]
      },
      "contentcomp": {
        "excl": [
          { "cat": ["IAB25"] },
          { "cat": ["IAB26"] }
        ]
      },
      "devicecomp": {
        "incl": [
          { "devicetype": 2 },
          { "devicetype": 1 }
        ],
        "excl": [
          { "devicetype": 3 },
          { "devicetype": 7 }
        ]
      },
      "sitecomp": {
        "incl": [
          {
            "domain": "therecipehub.com",
            "cat": ["IAB8"],
            "publisher": {
              "id": "pub-trh-001",
              "name": "The Recipe Hub",
              "domain": "therecipehub.com"
            }
          },
          {
            "domain": "homechefweekly.com",
            "cat": ["IAB8", "IAB8-12"],
            "publisher": {
              "id": "pub-hcw-001",
              "name": "Home Chef Weekly",
              "domain": "homechefweekly.com"
            }
          },
          {
            "domain": "mealinspo.com",
            "cat": ["IAB8", "IAB8-8"],
            "publisher": {
              "id": "pub-mi-001",
              "name": "Meal Inspo",
              "domain": "mealinspo.com"
            }
          }
        ]
      }
    }
  }
}
```

The buyer receives this push and stores the deal with `buyerstatus=0` (PENDING). Because this is a library deal with no seat restrictions, the buyer's system makes it available to all seats once accepted. In the baseline model, the buyer reviews the terms internally and communicates their verdict by updating their status on their side — no push to the seller is required.

---

### Step 2: Seller Polls Buyer's Endpoint (Accepted, Not Yet Live)

The seller periodically polls the buyer's endpoint (GET) to check whether the deal has been accepted. On this poll, the buyer has accepted — their system has updated `negotiationstatus` on `currentrevision` to ACCEPTED and promoted it to `liverevision`. The response includes the fully merged live state in the top-level `terms` and `inventory` fields, and `currentrevision` is absent (no pending proposal). Because this is the buyer's endpoint, the response includes a `seatstatuses` array with per-seat operational detail. At this point one seat has already opted into the library deal, skipping `status=0` (PENDING) and entering directly at `status=1` (NOT_STARTED).

**Request**
```
GET https://dsp.buyerco.com/deal-sync/v1/deals/deal-web-auto-q2-001
```

**Response**
```json
{
  "id": "deal-web-auto-q2-001",
  "sellerdealid": "PWG-2026-AUTO-887",
  "name": "In-Market Auto Intenders — Web (Library Deal)",
  "desc": "Evergreen multi-publisher web deal targeting in-market auto intenders via IAB Audience Taxonomy signals, served across cooking and recipe publisher inventory. Banner and outstream video. Publisher roster subject to change. Open to all buyer seats.",
  "origin": "adxchange.io",
  "seller": "premiumwebgroup.com",
  "created": "2026-03-24T08:00:00Z",
  "buyerstatus": 1,
  "adtypes": [1, 2],
  "auxdata": 2,
  "pubcount": 2,
  "dinventory": 2,
  "curation": {
    "curator": "premiumwebgroup.com",
    "cdealid": "PWG-CUR-2026-AUTO-887",
    "curfeetype": 3
  },
  "terms": {
    "startdate": "2026-04-01T00:00:00Z",
    "countries": ["USA"],
    "dealfloor": 4.50,
    "cur": "USD",
    "pricetype": 2,
    "guar": 0
  },
  "inventory": {
    "usercomp": {
      "incl": [
        {
          "name": "premiumwebgroup.com",
          "ext": { "segtax": 4 },
          "segment": [
            { "id": "284" },
            { "id": "285" },
            { "id": "287" }
          ]
        },
        {
          "name": "datapartner.com",
          "ext": { "segtax": 4 },
          "segment": [
            { "id": "284" },
            { "id": "286" }
          ]
        }
      ]
    },
    "contentcomp": {
      "excl": [
        { "cat": ["IAB25"] },
        { "cat": ["IAB26"] }
      ]
    },
    "devicecomp": {
      "incl": [
        { "devicetype": 2 },
        { "devicetype": 1 }
      ],
      "excl": [
        { "devicetype": 3 },
        { "devicetype": 7 }
      ]
    },
    "sitecomp": {
      "incl": [
        {
          "domain": "therecipehub.com",
          "cat": ["IAB8"],
          "publisher": {
            "id": "pub-trh-001",
            "name": "The Recipe Hub",
            "domain": "therecipehub.com"
          }
        },
        {
          "domain": "homechefweekly.com",
          "cat": ["IAB8", "IAB8-12"],
          "publisher": {
            "id": "pub-hcw-001",
            "name": "Home Chef Weekly",
            "domain": "homechefweekly.com"
          }
        },
        {
          "domain": "mealinspo.com",
          "cat": ["IAB8", "IAB8-8"],
          "publisher": {
            "id": "pub-mi-001",
            "name": "Meal Inspo",
            "domain": "mealinspo.com"
          }
        }
      ]
    }
  },
  "seatstatuses": [
    {
      "buyerseatid": "seat-ttd-main-001",
      "status": 1
    }
  ],
  "liverevision": {
    "revisionid": "f2e9c4b1-7a3d-4f8e-b6c2-9d5a1e7f4b08",
    "revisedate": "2026-03-24T08:00:00Z",
    "revisedby": {
      "partyid": "premiumwebgroup.com",
      "contactemail": "programmatic@premiumwebgroup.com",
      "role": 0
    },
    "negotiationstatus": 1,
    "comment": "Initial proposal. Evergreen library deal — audience defined via IAB Audience Taxonomy 1.1 in-market auto intender segments, overlaid on cooking and recipe publisher inventory. Publisher list subject to change; content and device dimensions are indicative. Open to all buyer seats."
  }
}
```

`buyerstatus=1` (NOT_STARTED) — the deal has been accepted but the flight start date of April 1 has not yet been reached. One seat (`seat-ttd-main-001`) has already opted into the library deal at `status=1` (NOT_STARTED), skipping the approval gate since library deals are open to all seats. The seller can begin preparing bid request targeting against `deal.id`. `liverevision` confirms the accepted terms; `currentrevision` is absent as there is no pending proposal.

---

### Step 3: Seller Polls Buyer's Endpoint (Live, Multiple Seats Active)

The seller polls the buyer's endpoint again after the deal's start date has passed. The deal is now live: `buyerstatus=2` (LIVE). Three buyer seats have opted into the library deal. Two are actively delivering (`status=4`, ACTIVE), and one has been paused by the buyer (`status=5`, PAUSED). Because library deals are open to all seats, new seats may appear in `seatstatuses` at any time without requiring seller approval.

**Request**
```
GET https://dsp.buyerco.com/deal-sync/v1/deals/deal-web-auto-q2-001
```

**Response**
```json
{
  "id": "deal-web-auto-q2-001",
  "sellerdealid": "PWG-2026-AUTO-887",
  "name": "In-Market Auto Intenders — Web (Library Deal)",
  "desc": "Evergreen multi-publisher web deal targeting in-market auto intenders via IAB Audience Taxonomy signals, served across cooking and recipe publisher inventory. Banner and outstream video. Publisher roster subject to change. Open to all buyer seats.",
  "origin": "adxchange.io",
  "seller": "premiumwebgroup.com",
  "created": "2026-03-24T08:00:00Z",
  "buyerstatus": 2,
  "adtypes": [1, 2],
  "auxdata": 2,
  "pubcount": 2,
  "dinventory": 2,
  "curation": {
    "curator": "premiumwebgroup.com",
    "cdealid": "PWG-CUR-2026-AUTO-887",
    "curfeetype": 3
  },
  "terms": {
    "startdate": "2026-04-01T00:00:00Z",
    "countries": ["USA"],
    "dealfloor": 4.50,
    "cur": "USD",
    "pricetype": 2,
    "guar": 0
  },
  "inventory": {
    "usercomp": {
      "incl": [
        {
          "name": "premiumwebgroup.com",
          "ext": { "segtax": 4 },
          "segment": [
            { "id": "284" },
            { "id": "285" },
            { "id": "287" }
          ]
        },
        {
          "name": "datapartner.com",
          "ext": { "segtax": 4 },
          "segment": [
            { "id": "284" },
            { "id": "286" }
          ]
        }
      ]
    },
    "contentcomp": {
      "excl": [
        { "cat": ["IAB25"] },
        { "cat": ["IAB26"] }
      ]
    },
    "devicecomp": {
      "incl": [
        { "devicetype": 2 },
        { "devicetype": 1 }
      ],
      "excl": [
        { "devicetype": 3 },
        { "devicetype": 7 }
      ]
    },
    "sitecomp": {
      "incl": [
        {
          "domain": "therecipehub.com",
          "cat": ["IAB8"],
          "publisher": {
            "id": "pub-trh-001",
            "name": "The Recipe Hub",
            "domain": "therecipehub.com"
          }
        },
        {
          "domain": "homechefweekly.com",
          "cat": ["IAB8", "IAB8-12"],
          "publisher": {
            "id": "pub-hcw-001",
            "name": "Home Chef Weekly",
            "domain": "homechefweekly.com"
          }
        },
        {
          "domain": "mealinspo.com",
          "cat": ["IAB8", "IAB8-8"],
          "publisher": {
            "id": "pub-mi-001",
            "name": "Meal Inspo",
            "domain": "mealinspo.com"
          }
        }
      ]
    }
  },
  "seatstatuses": [
    {
      "buyerseatid": "seat-ttd-main-001",
      "status": 4
    },
    {
      "buyerseatid": "seat-ttd-west-047",
      "status": 4
    },
    {
      "buyerseatid": "seat-ttd-east-012",
      "status": 5
    }
  ],
  "liverevision": {
    "revisionid": "f2e9c4b1-7a3d-4f8e-b6c2-9d5a1e7f4b08",
    "revisedate": "2026-03-24T08:00:00Z",
    "revisedby": {
      "partyid": "premiumwebgroup.com",
      "contactemail": "programmatic@premiumwebgroup.com",
      "role": 0
    },
    "negotiationstatus": 1,
    "comment": "Initial proposal. Evergreen library deal — audience defined via IAB Audience Taxonomy 1.1 in-market auto intender segments, overlaid on cooking and recipe publisher inventory. Publisher list subject to change; content and device dimensions are indicative. Open to all buyer seats."
  }
}
```

`buyerstatus=2` (LIVE) — the deal is actively delivering. Three seats have opted in: `seat-ttd-main-001` and `seat-ttd-west-047` are at `status=4` (ACTIVE), meaning they are bidding and winning impressions; `seat-ttd-east-012` is at `status=5` (PAUSED), indicating the buyer has temporarily paused that seat's participation. The seller can use this information to understand demand distribution across the library deal's buyer base. Because this is an evergreen deal with no `enddate`, it will remain live indefinitely until one party moves it to a terminal state.

---

<a name="scenario-2-seller-initiated-ctv-deal-with-buyer-revision"></a>
## Scenario 2: Seller-Initiated CTV Deal with Buyer Revision

A seller (Meridian SSP, `meridian-ssp.tv`) initiates a premium CTV deal covering drama and live sports programming across two streaming apps. The buyer (`buyerco.com`) accepts the initial proposal, then proposes a revision to expand the geographic scope to Canada with a French-language exclusion. Both parties have agreed to support the optional bidirectional model.

---

### Step 1: Seller Creates the Deal

The seller initiates a new deal by pushing a Deal object to the buyer's push endpoint. Because no revision has yet been accepted, there is no `liverevision` and the top-level `terms` and `inventory` fields are omitted. The full deal specification lives in `currentrevision`, which must be a complete description of the deal terms (not a delta) until a `liverevision` is established.

**Request**
```
POST https://dsp.buyerco.com/deal-sync/v1/push
Content-Type: application/json
```

**Payload**
```json
{
  "id": "deal-ctv-q3-premium-001",
  "sellerdealid": "MER-2026-CTV-4421",
  "name": "Q3 2026 Premium CTV — Drama & Live Sports",
  "desc": "Premium CTV inventory across Apex Streaming and VuePlex Free TV. Professionally produced drama and live sports on connected TV and set-top box devices, US only.",
  "origin": "meridian-ssp.tv",
  "seller": "apexstreaming.tv",
  "created": "2026-03-20T09:00:00Z",
  "sellerstatus": 0,
  "currentrevision": {
    "revisionid": "a4c2f1e8-3b7d-4a9f-8c5e-1d6b2f0e3a47",
    "revisedate": "2026-03-20T09:00:00Z",
    "revisedby": {
      "partyid": "meridian-ssp.tv",
      "contactemail": "deals@meridian-ssp.tv",
      "role": 0
    },
    "negotiationstatus": 0,
    "comment": "Initial proposal for Q3 premium CTV package. Floor reflects blended drama/sports CPM based on Q2 actuals.",
    "adtypes": [2],
    "auxdata": 3,
    "pubcount": 2,
    "dinventory": 1,
    "terms": {
      "startdate": "2026-07-01T00:00:00Z",
      "enddate": "2026-09-30T23:59:59Z",
      "countries": ["USA"],
      "dealfloor": 12.50,
      "cur": "USD",
      "pricetype": 2,
      "guar": 0
    },
    "inventory": {
      "contentcomp": {
        "incl": [
          {
            "prodq": 1,
            "cat": ["IAB1-7"],
            "genre": "Drama",
            "livestream": 0,
            "language": "en",
            "contentrating": "TV-14"
          },
          {
            "prodq": 1,
            "cat": ["IAB17"],
            "genre": "Sports",
            "livestream": 1,
            "language": "en"
          }
        ],
        "excl": [
          { "cat": ["IAB12"] },
          { "prodq": 3 }
        ]
      },
      "devicecomp": {
        "incl": [
          { "devicetype": 3 },
          { "devicetype": 7 }
        ],
        "excl": [
          { "devicetype": 4 },
          { "devicetype": 2 }
        ]
      },
      "appcomp": {
        "incl": [
          {
            "name": "Apex Streaming",
            "bundle": "com.apexstreaming.ctv",
            "domain": "apexstreaming.tv",
            "cat": ["IAB1-7", "IAB17"],
            "publisher": {
              "id": "pub-apex-001",
              "name": "Apex Streaming, Inc.",
              "domain": "apexstreaming.tv"
            }
          },
          {
            "name": "VuePlex Free TV",
            "bundle": "com.vueplex.tv",
            "domain": "vueplex.tv",
            "cat": ["IAB1-7", "IAB17"],
            "publisher": {
              "id": "pub-vueplex-001",
              "name": "VuePlex Media Group",
              "domain": "vueplex.tv"
            }
          }
        ],
        "excl": [
          {
            "bundle": "com.apexstreaming.kids",
            "publisher": { "domain": "apexstreaming.tv" }
          },
          {
            "bundle": "com.vueplex.kids",
            "publisher": { "domain": "vueplex.tv" }
          }
        ]
      }
    }
  }
}
```

The buyer receives this push and stores the deal with `buyerstatus=0` (PENDING).

---

### Step 2: Buyer Accepts the Initial Proposal

After reviewing the terms, the buyer accepts by pushing a DealResponse to the seller's push endpoint. The `revisionid` must match the `currentrevision.revisionid` from the seller's initial push. On receipt, the seller validates the match, transitions `negotiationstatus` on the initial revision to ACCEPTED, promotes it to `liverevision`, and advances both `sellerstatus` and `buyerstatus` to `1` (NOT_STARTED) or `2` (LIVE) depending on whether the flight start date has been reached.

**Request**
```
POST https://meridian-ssp.tv/deal-sync/v1/push
Content-Type: application/json
```

**Payload**
```json
{
  "dealid": "deal-ctv-q3-premium-001",
  "revisionid": "a4c2f1e8-3b7d-4a9f-8c5e-1d6b2f0e3a47",
  "negotiationstatus": 1,
  "respondedby": {
    "partyid": "dsp-seat-ttd-001",
    "contactemail": "trader@buyerco.com",
    "role": 1
  },
  "responsedate": "2026-03-21T10:15:00Z",
  "comment": "Terms accepted. Trafficking against this deal for Q3."
}
```

---

### Step 3: Buyer Proposes a Revision

The deal has been accepted and is live. The buyer wants to expand the geographic scope to include Canada, but also needs to exclude French-language content to match their English-only creative. They push a Deal object with a delta `currentrevision` to the seller's push endpoint.

Because a `liverevision` now exists, `currentrevision` carries only the fields that differ from it. `terms` is included with the updated `countries` array. `inventory` is included with an updated `contentcomp` — `devicecomp` and `appcomp` are unchanged and omitted.

Note that array fields replace in full rather than merge. This means the `contentcomp.excl` array must restate the existing exclusions (`cat: ["IAB12"]` and `prodq: 3`) alongside the new `language: "fr"` entry. Omitting them would cause the receiver to drop them from the live terms upon acceptance.

**Request**
```
POST https://meridian-ssp.tv/deal-sync/v1/push
Content-Type: application/json
```

**Payload**
```json
{
  "id": "deal-ctv-q3-premium-001",
  "buyerstatus": 2,
  "currentrevision": {
    "revisionid": "c9f7b3a2-1e4d-4c8f-9b2a-5e7d1f3c6a09",
    "revisedate": "2026-04-15T11:22:00Z",
    "revisedby": {
      "partyid": "dsp-seat-ttd-001",
      "contactemail": "trader@buyerco.com",
      "role": 1
    },
    "negotiationstatus": 0,
    "comment": "Requesting Canada expansion. Adding French-language exclusion to maintain brand suitability for English-only creative.",
    "terms": {
      "startdate": "2026-07-01T00:00:00Z",
      "enddate": "2026-09-30T23:59:59Z",
      "countries": ["USA", "CAN"],
      "dealfloor": 12.50,
      "cur": "USD",
      "pricetype": 2,
      "guar": 0
    },
    "inventory": {
      "contentcomp": {
        "incl": [
          {
            "prodq": 1,
            "cat": ["IAB1-7"],
            "genre": "Drama",
            "livestream": 0,
            "language": "en",
            "contentrating": "TV-14"
          },
          {
            "prodq": 1,
            "cat": ["IAB17"],
            "genre": "Sports",
            "livestream": 1,
            "language": "en"
          }
        ],
        "excl": [
          { "cat": ["IAB12"] },
          { "prodq": 3 },
          { "language": "fr" }
        ]
      }
    }
  }
}
```

The seller receives this push and sets the deal's `currentrevision` to this proposed revision. The deal remains live under the previously accepted `liverevision` terms while the seller reviews the proposal.

---

### Step 4: Seller Accepts the Revision

The seller agrees to the changes and pushes a DealResponse to the buyer's push endpoint. A DealResponse carries no deal term fields — only the verdict (`negotiationstatus`) and a reference to the revision being acted upon (`revisionid`). The buyer validates that the referenced `revisionid` matches the `currentrevision.revisionid` they proposed. On a match, `currentrevision` transitions to ACCEPTED and becomes the new `liverevision`.

**Request**
```
POST https://dsp.buyerco.com/deal-sync/v1/push
Content-Type: application/json
```

**Payload**
```json
{
  "dealid": "deal-ctv-q3-premium-001",
  "revisionid": "c9f7b3a2-1e4d-4c8f-9b2a-5e7d1f3c6a09",
  "negotiationstatus": 1,
  "respondedby": {
    "partyid": "meridian-ssp.tv",
    "contactemail": "deals@meridian-ssp.tv",
    "role": 0
  },
  "responsedate": "2026-04-15T14:05:00Z",
  "comment": "Accepted. Canada geo and French-language exclusion confirmed."
}
```

---

### Step 5: Buyer Polls Seller's Endpoint

The buyer queries the seller's endpoint (GET) to confirm the current state of the deal. Because a `liverevision` is now established and there is no pending proposal, `currentrevision` is absent — `liverevision` alone fully describes the current terms. The top-level `terms` and `inventory` reflect the fully merged live state.

**Request**
```
GET https://meridian-ssp.tv/deal-sync/v1/deals/deal-ctv-q3-premium-001
```

**Response**
```json
{
  "id": "deal-ctv-q3-premium-001",
  "sellerdealid": "MER-2026-CTV-4421",
  "name": "Q3 2026 Premium CTV — Drama & Live Sports",
  "desc": "Premium CTV inventory across Apex Streaming and VuePlex Free TV. Professionally produced drama and live sports on connected TV and set-top box devices, US only.",
  "origin": "meridian-ssp.tv",
  "seller": "apexstreaming.tv",
  "created": "2026-03-20T09:00:00Z",
  "sellerstatus": 2,
  "adtypes": [2],
  "auxdata": 3,
  "pubcount": 2,
  "dinventory": 1,
  "terms": {
    "startdate": "2026-07-01T00:00:00Z",
    "enddate": "2026-09-30T23:59:59Z",
    "countries": ["USA", "CAN"],
    "dealfloor": 12.50,
    "cur": "USD",
    "pricetype": 2,
    "guar": 0
  },
  "inventory": {
    "contentcomp": {
      "incl": [
        {
          "prodq": 1,
          "cat": ["IAB1-7"],
          "genre": "Drama",
          "livestream": 0,
          "language": "en",
          "contentrating": "TV-14"
        },
        {
          "prodq": 1,
          "cat": ["IAB17"],
          "genre": "Sports",
          "livestream": 1,
          "language": "en"
        }
      ],
      "excl": [
        { "cat": ["IAB12"] },
        { "prodq": 3 },
        { "language": "fr" }
      ]
    },
    "devicecomp": {
      "incl": [
        { "devicetype": 3 },
        { "devicetype": 7 }
      ],
      "excl": [
        { "devicetype": 4 },
        { "devicetype": 2 }
      ]
    },
    "appcomp": {
      "incl": [
        {
          "name": "Apex Streaming",
          "bundle": "com.apexstreaming.ctv",
          "domain": "apexstreaming.tv",
          "cat": ["IAB1-7", "IAB17"],
          "publisher": {
            "id": "pub-apex-001",
            "name": "Apex Streaming, Inc.",
            "domain": "apexstreaming.tv"
          }
        },
        {
          "name": "VuePlex Free TV",
          "bundle": "com.vueplex.tv",
          "domain": "vueplex.tv",
          "cat": ["IAB1-7", "IAB17"],
          "publisher": {
            "id": "pub-vueplex-001",
            "name": "VuePlex Media Group",
            "domain": "vueplex.tv"
          }
        }
      ],
      "excl": [
        {
          "bundle": "com.apexstreaming.kids",
          "publisher": { "domain": "apexstreaming.tv" }
        },
        {
          "bundle": "com.vueplex.kids",
          "publisher": { "domain": "vueplex.tv" }
        }
      ]
    }
  },
  "liverevision": {
    "revisionid": "c9f7b3a2-1e4d-4c8f-9b2a-5e7d1f3c6a09",
    "revisedate": "2026-04-15T11:22:00Z",
    "revisedby": {
      "partyid": "dsp-seat-ttd-001",
      "contactemail": "trader@buyerco.com",
      "role": 1
    },
    "negotiationstatus": 1,
    "comment": "Requesting Canada expansion. Adding French-language exclusion to maintain brand suitability for English-only creative."
  }
}
```

The buyer can confirm `sellerstatus=2` (LIVE) and traffic against the deal's terms with confidence that `liverevision` represents the fully settled state.

---

<a name="scenario-3-buyer-initiated-mobile-gaming-deal-with-seller-revision"></a>
## Scenario 3: Buyer-Initiated Mobile Gaming Deal with Seller Revision

A buyer (`buyerco.com`) initiates a deal targeting premium mobile gaming inventory across three apps from different publishers. The deal specifies an exhaustive app list and restricts to mobile devices only. The seller (GameGrid SSP, `gamegrid-ssp.com`) accepts, then proposes a revision to expand the deal into console gaming supply on Xbox and PlayStation. The buyer accepts the expansion. Part-way into delivery, the seller pauses the deal, and the buyer subsequently cancels it.

This scenario uses the **bidirectional model**: both parties push to each other's endpoints.

---

### Step 1: Buyer Creates the Deal

The buyer initiates a new deal by pushing a Deal object to the seller's push endpoint. Per the spec, the buyer proposes a value for `id`, but it is formally confirmed by the seller upon acceptance since the seller (SSP) controls bid request construction. The buyer populates `buyerdealid` with their own internal reference. `buyerstatus=0` (PENDING) reflects the buyer's view; `sellerstatus` is omitted since the buyer does not know the seller's state yet.

Because no revision has been accepted, `currentrevision` carries the full deal specification. The `appcomp` lists all three apps that make up the complete set of inventory for the deal.

**Request**
```
POST https://gamegrid-ssp.com/deal-sync/v1/push
Content-Type: application/json
```

**Payload**
```json
{
  "id": "deal-game-mobile-q3-001",
  "buyerdealid": "BUYER-2026-GAME-7710",
  "name": "Q3 2026 Premium Mobile Gaming — Puzzle, Action & Racing",
  "desc": "Buyer-initiated deal targeting premium mobile gaming inventory across three titles. Video and banner. US and UK. Mobile devices only.",
  "origin": "gamegrid-ssp.com",
  "seller": "gamegrid-ssp.com",
  "created": "2026-05-10T14:00:00Z",
  "buyerstatus": 0,
  "currentrevision": {
    "revisionid": "b1d4e7a3-9c2f-4b8e-a5d1-7f3c6e0b9a24",
    "revisedate": "2026-05-10T14:00:00Z",
    "revisedby": {
      "partyid": "dsp-seat-ttd-001",
      "contactemail": "trader@buyerco.com",
      "role": 1
    },
    "negotiationstatus": 0,
    "comment": "Buyer-initiated deal for Q3 mobile gaming package. Exhaustive app list — three titles across puzzle, action, and racing genres. Mobile only.",
    "adtypes": [1, 2],
    "auxdata": 3,
    "pubcount": 2,
    "dinventory": 1,
    "terms": {
      "startdate": "2026-07-01T00:00:00Z",
      "enddate": "2026-09-30T23:59:59Z",
      "countries": ["USA", "GBR"],
      "dealfloor": 8.00,
      "cur": "USD",
      "pricetype": 2,
      "guar": 0
    },
    "inventory": {
      "contentcomp": {
        "excl": [
          { "cat": ["IAB25"] },
          { "cat": ["IAB26"] }
        ]
      },
      "devicecomp": {
        "incl": [
          { "devicetype": 4 },
          { "devicetype": 5 }
        ],
        "excl": [
          { "devicetype": 2 },
          { "devicetype": 3 }
        ]
      },
      "appcomp": {
        "incl": [
          {
            "name": "Puzzle Quest Saga",
            "bundle": "com.puzzlecraft.saga",
            "domain": "puzzlecraftgames.com",
            "cat": ["IAB9-30"],
            "publisher": {
              "id": "pub-pcg-001",
              "name": "Puzzle Craft Games",
              "domain": "puzzlecraftgames.com"
            }
          },
          {
            "name": "Battle Royale Arena",
            "bundle": "com.stormgate.battlearena",
            "domain": "stormgatestudios.com",
            "cat": ["IAB9-30"],
            "publisher": {
              "id": "pub-sg-001",
              "name": "Stormgate Studios",
              "domain": "stormgatestudios.com"
            }
          },
          {
            "name": "Speed Rivals Racing",
            "bundle": "com.driftworks.speedrivals",
            "domain": "driftworks.io",
            "cat": ["IAB9-30"],
            "publisher": {
              "id": "pub-dw-001",
              "name": "Driftworks Interactive",
              "domain": "driftworks.io"
            }
          }
        ]
      }
    }
  }
}
```

The seller receives this push and stores the deal with `sellerstatus=0` (PENDING).

---

### Step 2: Seller Accepts the Initial Proposal

The seller reviews the terms and accepts by pushing a DealResponse to the buyer's push endpoint. By accepting, the seller confirms the buyer-proposed `id` as the canonical deal identifier that will appear in bid requests.

**Request**
```
POST https://dsp.buyerco.com/deal-sync/v1/push
Content-Type: application/json
```

**Payload**
```json
{
  "dealid": "deal-game-mobile-q3-001",
  "revisionid": "b1d4e7a3-9c2f-4b8e-a5d1-7f3c6e0b9a24",
  "negotiationstatus": 1,
  "respondedby": {
    "partyid": "gamegrid-ssp.com",
    "contactemail": "deals@gamegrid-ssp.com",
    "role": 0
  },
  "responsedate": "2026-05-11T09:30:00Z",
  "comment": "Accepted. All three titles confirmed in our system. Deal ID locked for bid requests."
}
```

Both `sellerstatus` and `buyerstatus` transition to `1` (NOT_STARTED). When the July 1 flight date arrives, both transition to `2` (LIVE).

---

### Step 3: Seller Proposes a Revision (Console Expansion)

The deal is live. The seller sees an opportunity to expand the deal into console gaming — two of the three titles (Battle Royale Arena and Speed Rivals Racing) are also available on Xbox and PlayStation. The seller pushes a Deal object with a delta `currentrevision` to the buyer's push endpoint.

The revision updates two composition dimensions. `devicecomp` adds `devicetype: 6` (Connected Device / Game Console) to the inclusion list. `appcomp` adds two console app entries. Because array fields replace in full, both arrays restate the existing mobile entries alongside the new console additions. `contentcomp` is unchanged and omitted from the delta.

**Request**
```
POST https://dsp.buyerco.com/deal-sync/v1/push
Content-Type: application/json
```

**Payload**
```json
{
  "id": "deal-game-mobile-q3-001",
  "sellerstatus": 2,
  "currentrevision": {
    "revisionid": "e3a8f2c1-5d7b-4e9a-b4f6-2c8d0a1e5f37",
    "revisedate": "2026-07-18T16:45:00Z",
    "revisedby": {
      "partyid": "gamegrid-ssp.com",
      "contactemail": "deals@gamegrid-ssp.com",
      "role": 0
    },
    "negotiationstatus": 0,
    "comment": "Proposing console expansion. Battle Royale Arena and Speed Rivals Racing are now available on Xbox and PlayStation. Adding console device type and console app bundles.",
    "inventory": {
      "devicecomp": {
        "incl": [
          { "devicetype": 4 },
          { "devicetype": 5 },
          { "devicetype": 6 }
        ],
        "excl": [
          { "devicetype": 2 },
          { "devicetype": 3 }
        ]
      },
      "appcomp": {
        "incl": [
          {
            "name": "Puzzle Quest Saga",
            "bundle": "com.puzzlecraft.saga",
            "domain": "puzzlecraftgames.com",
            "cat": ["IAB9-30"],
            "publisher": {
              "id": "pub-pcg-001",
              "name": "Puzzle Craft Games",
              "domain": "puzzlecraftgames.com"
            }
          },
          {
            "name": "Battle Royale Arena",
            "bundle": "com.stormgate.battlearena",
            "domain": "stormgatestudios.com",
            "cat": ["IAB9-30"],
            "publisher": {
              "id": "pub-sg-001",
              "name": "Stormgate Studios",
              "domain": "stormgatestudios.com"
            }
          },
          {
            "name": "Speed Rivals Racing",
            "bundle": "com.driftworks.speedrivals",
            "domain": "driftworks.io",
            "cat": ["IAB9-30"],
            "publisher": {
              "id": "pub-dw-001",
              "name": "Driftworks Interactive",
              "domain": "driftworks.io"
            }
          },
          {
            "name": "Battle Royale Arena — Console",
            "bundle": "com.stormgate.battlearena.console",
            "domain": "stormgatestudios.com",
            "cat": ["IAB9-30"],
            "publisher": {
              "id": "pub-sg-001",
              "name": "Stormgate Studios",
              "domain": "stormgatestudios.com"
            }
          },
          {
            "name": "Speed Rivals Racing — Console",
            "bundle": "com.driftworks.speedrivals.console",
            "domain": "driftworks.io",
            "cat": ["IAB9-30"],
            "publisher": {
              "id": "pub-dw-001",
              "name": "Driftworks Interactive",
              "domain": "driftworks.io"
            }
          }
        ]
      }
    }
  }
}
```

The buyer receives this push. The deal remains live under the original `liverevision` terms (mobile only) while the buyer evaluates the console expansion.

---

### Step 4: Buyer Accepts the Revision

The buyer agrees to the console expansion and pushes a DealResponse to the seller's push endpoint. The accepted revision becomes the new `liverevision`, and the deal now covers both mobile and console inventory.

**Request**
```
POST https://gamegrid-ssp.com/deal-sync/v1/push
Content-Type: application/json
```

**Payload**
```json
{
  "dealid": "deal-game-mobile-q3-001",
  "revisionid": "e3a8f2c1-5d7b-4e9a-b4f6-2c8d0a1e5f37",
  "negotiationstatus": 1,
  "respondedby": {
    "partyid": "dsp-seat-ttd-001",
    "contactemail": "trader@buyerco.com",
    "role": 1
  },
  "responsedate": "2026-07-19T10:00:00Z",
  "comment": "Console expansion accepted. Updating targeting to include Xbox and PlayStation inventory."
}
```

---

### Step 5: Seller Pauses the Deal

Several weeks into the expanded deal, the seller encounters a supply quality issue with one of the console app bundles and decides to pause the deal while they investigate. The seller pushes an updated Deal object to the buyer's push endpoint with `sellerstatus=4` (PAUSED).

Because `sellerstatus=PAUSED` is the effective deal-dark signal — the seller controls bid request delivery — the deal is no longer eligible for auction regardless of the buyer's status. No revision is involved; this is a lifecycle state change only.

**Request**
```
POST https://dsp.buyerco.com/deal-sync/v1/push
Content-Type: application/json
```

**Payload**
```json
{
  "id": "deal-game-mobile-q3-001",
  "sellerstatus": 4
}
```

The buyer receives this push and updates their record to reflect that the seller has paused the deal. The buyer's own `buyerstatus` may remain `2` (LIVE) — the buyer is still willing to traffic, but no bid requests will arrive while the seller side is paused.

---

### Step 6: Buyer Cancels the Deal

After a week with the deal paused and no communication from the seller about resuming, the buyer decides to cancel. The buyer pushes an updated Deal object to the seller's push endpoint with `buyerstatus=6` (CANCELED).

CANCELED is a terminal state. Per the coordination rules, the seller should mirror it by setting `sellerstatus=7` (CANCELED) upon receiving this push. To reactivate this inventory relationship, the parties would need to create a new deal.

**Request**
```
POST https://gamegrid-ssp.com/deal-sync/v1/push
Content-Type: application/json
```

**Payload**
```json
{
  "id": "deal-game-mobile-q3-001",
  "buyerstatus": 6
}
```

The seller receives this push, validates the terminal state, and transitions `sellerstatus` from `4` (PAUSED) to `7` (CANCELED). The deal is now closed on both sides.
