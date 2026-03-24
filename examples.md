# Deal Sync API v1.1 — Example Workflows

## Table of Contents

- [Scenario 1: Saleshouse Audience Deal Across Multiple Web Publishers](#scenario-1-saleshouse-audience-deal-across-multiple-web-publishers)
  - [Step 1: Saleshouse Creates the Deal](#step-1-saleshouse-creates-the-deal)
  - [Step 2: Seller Polls Buyer's Status Endpoint](#step-2-seller-polls-buyers-status-endpoint)
- [Scenario 2: Seller-Initiated CTV Deal with Buyer Revision](#scenario-2-seller-initiated-ctv-deal-with-buyer-revision)
  - [Step 1: Seller Creates the Deal](#step-1-seller-creates-the-deal)
  - [Step 2: Buyer Accepts the Initial Proposal](#step-2-buyer-accepts-the-initial-proposal)
  - [Step 3: Buyer Proposes a Revision](#step-3-buyer-proposes-a-revision)
  - [Step 4: Seller Accepts the Revision](#step-4-seller-accepts-the-revision)
  - [Step 5: Buyer Polls Seller's Status Endpoint](#step-5-buyer-polls-sellers-status-endpoint)

---

<a name="scenario-1-saleshouse-audience-deal-across-multiple-web-publishers"></a>
## Scenario 1: Saleshouse Audience Deal Across Multiple Web Publishers

A technical saleshouse (PremiumWeb Group, `premiumwebgroup.com`) represents three web publishers and packages their combined inventory against an in-market auto intender audience defined using IAB Audience Taxonomy 1.1 signals. The deal covers banner and outstream video creatives. Because the saleshouse may adjust its publisher roster over the flight, inventory is dynamic (`dinventory=2`). The saleshouse charges a CPM curation fee for the audience service.

This scenario uses the **baseline seller-push model**: the saleshouse pushes the deal to the buyer's endpoint, then polls the buyer's status endpoint to learn when the buyer has accepted. The buyer does not implement a push endpoint — they communicate acceptance by updating the deal state on their side, which the seller discovers on its next poll.

---

### Step 1: Saleshouse Creates the Deal

The saleshouse pushes the initial deal to the buyer's push endpoint. The audience composition (`usercomp`) is the primary targeting signal — content and device composition are included but play a supporting role (brand safety and format eligibility respectively). Because no revision has yet been accepted, `currentrevision` carries the full deal specification.

`sitecomp` is included with `fidelity=1` (indicative) to identify the current publisher roster and support supply chain authorization, with the understanding that the list may change over the flight.

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
  "name": "Q2 2026 In-Market Auto Intenders — Web",
  "desc": "Multi-publisher web deal targeting in-market auto intenders via IAB Audience Taxonomy signals. Banner and outstream video. Publisher roster subject to change over flight.",
  "origin": "adxchange.io",
  "seller": "premiumwebgroup.com",
  "created": "2026-03-24T08:00:00Z",
  "dealstatus": 0,
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
    "comment": "Initial proposal. Audience defined via IAB Audience Taxonomy 1.1 in-market auto segments across three publisher properties. Publisher list subject to change; content and device dimensions are indicative.",
    "adtypes": [1, 2],
    "auxdata": 2,
    "pubcount": 2,
    "dinventory": 2,
    "terms": {
      "startdate": "2026-04-01T00:00:00Z",
      "enddate": "2026-06-30T23:59:59Z",
      "countries": ["USA"],
      "dealfloor": 4.50,
      "cur": "USD",
      "pricetype": 2,
      "guar": 0
    },
    "inventory": {
      "usercomp": {
        "fidelity": 2,
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
        "fidelity": 1,
        "excl": [
          { "cat": ["IAB25"] },
          { "cat": ["IAB26"] }
        ]
      },
      "devicecomp": {
        "fidelity": 2,
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
        "fidelity": 1,
        "incl": [
          {
            "domain": "autolifemedia.com",
            "cat": ["IAB2"],
            "publisher": {
              "id": "pub-alm-001",
              "name": "AutoLife Media",
              "domain": "autolifemedia.com"
            }
          },
          {
            "domain": "familywheels.com",
            "cat": ["IAB2", "IAB25-2"],
            "publisher": {
              "id": "pub-fw-001",
              "name": "Family Wheels",
              "domain": "familywheels.com"
            }
          },
          {
            "domain": "consumerfirst.com",
            "cat": ["IAB2", "IAB22"],
            "publisher": {
              "id": "pub-cf-001",
              "name": "ConsumerFirst",
              "domain": "consumerfirst.com"
            }
          }
        ]
      }
    }
  }
}
```

The buyer receives this push and stores the deal with `dealstatus=0` (PENDING_ACCEPTANCE). In the baseline model, the buyer reviews the terms internally and communicates their verdict by updating the deal state on their side — no push to the seller is required.

---

### Step 2: Seller Polls Buyer's Status Endpoint

The seller periodically polls the buyer's status endpoint to check whether the deal has been accepted. On this poll, the buyer has accepted — their system has updated `negotiationstatus` on `currentrevision` to ACCEPTED and promoted it to `liverevision`. The response includes the fully merged live state in the top-level `terms` and `inventory` fields, and `currentrevision` is absent (no pending proposal).

**Request**
```
GET https://dsp.buyerco.com/deal-sync/v1/deals/deal-web-auto-q2-001
```

**Response**
```json
{
  "id": "deal-web-auto-q2-001",
  "sellerdealid": "PWG-2026-AUTO-887",
  "name": "Q2 2026 In-Market Auto Intenders — Web",
  "desc": "Multi-publisher web deal targeting in-market auto intenders via IAB Audience Taxonomy signals. Banner and outstream video. Publisher roster subject to change over flight.",
  "origin": "adxchange.io",
  "seller": "premiumwebgroup.com",
  "created": "2026-03-24T08:00:00Z",
  "dealstatus": 1,
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
    "enddate": "2026-06-30T23:59:59Z",
    "countries": ["USA"],
    "dealfloor": 4.50,
    "cur": "USD",
    "pricetype": 2,
    "guar": 0
  },
  "inventory": {
    "usercomp": {
      "fidelity": 2,
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
      "fidelity": 1,
      "excl": [
        { "cat": ["IAB25"] },
        { "cat": ["IAB26"] }
      ]
    },
    "devicecomp": {
      "fidelity": 2,
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
      "fidelity": 1,
      "incl": [
        {
          "domain": "autolifemedia.com",
          "cat": ["IAB2"],
          "publisher": {
            "id": "pub-alm-001",
            "name": "AutoLife Media",
            "domain": "autolifemedia.com"
          }
        },
        {
          "domain": "familywheels.com",
          "cat": ["IAB2", "IAB25-2"],
          "publisher": {
            "id": "pub-fw-001",
            "name": "Family Wheels",
            "domain": "familywheels.com"
          }
        },
        {
          "domain": "consumerfirst.com",
          "cat": ["IAB2", "IAB22"],
          "publisher": {
            "id": "pub-cf-001",
            "name": "ConsumerFirst",
            "domain": "consumerfirst.com"
          }
        }
      ]
    }
  },
  "liverevision": {
    "revisionid": "f2e9c4b1-7a3d-4f8e-b6c2-9d5a1e7f4b08",
    "revisedate": "2026-03-24T08:00:00Z",
    "revisedby": {
      "partyid": "premiumwebgroup.com",
      "contactemail": "programmatic@premiumwebgroup.com",
      "role": 0
    },
    "negotiationstatus": 1,
    "comment": "Initial proposal. Audience defined via IAB Audience Taxonomy 1.1 in-market auto segments across three publisher properties. Publisher list subject to change; content and device dimensions are indicative."
  }
}
```

`dealstatus=1` (NOT_STARTED) — the deal has been accepted but the flight start date of April 1 has not yet been reached. The seller can begin preparing bid request targeting against `deal.id`. `liverevision` confirms the accepted terms; `currentrevision` is absent as there is no pending proposal.

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
  "dealstatus": 0,
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
        "fidelity": 1,
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
        "fidelity": 2,
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
        "fidelity": 2,
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

The buyer receives this push and stores the deal with `dealstatus=0` (PENDING_ACCEPTANCE).

---

### Step 2: Buyer Accepts the Initial Proposal

After reviewing the terms, the buyer accepts by pushing a DealResponse to the seller's push endpoint. The `revisionid` must match the `currentrevision.revisionid` from the seller's initial push. On receipt, the seller validates the match, transitions `negotiationstatus` on the initial revision to ACCEPTED, promotes it to `liverevision`, and advances `dealstatus` to `1` (NOT_STARTED) or `2` (LIVE) depending on whether the flight start date has been reached.

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
  "dealstatus": 2,
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
        "fidelity": 1,
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

### Step 5: Buyer Polls Seller's Status Endpoint

The buyer queries the seller's status endpoint to confirm the current state of the deal. Because a `liverevision` is now established and there is no pending proposal, `currentrevision` is absent — `liverevision` alone fully describes the current terms. The top-level `terms` and `inventory` reflect the fully merged live state.

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
  "dealstatus": 2,
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
      "fidelity": 1,
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
      "fidelity": 2,
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
      "fidelity": 2,
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

The buyer can confirm `dealstatus=2` (LIVE) and traffic against the deal's terms with confidence that `liverevision` represents the fully settled state.
