# Deal Sync API v1.1 — Example Workflows

The following examples illustrate a complete deal lifecycle between a seller (Meridian SSP, `meridian-ssp.tv`) and a buyer (`buyerco.com`). Both parties have agreed to support the optional bidirectional model.

The deal covers premium CTV inventory — drama and live sports programming — across two streaming apps on connected TV and set-top box devices in the US.

---

## Step 1: Seller Creates the Deal

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

## Step 2: Buyer Accepts the Initial Proposal

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

## Step 3: Buyer Proposes a Revision

The deal has been accepted and is live. The buyer wants to expand the geographic scope to include Canada, but also needs to exclude French-language content to match their English-only creative. They push a Deal object with a delta `currentrevision` to the seller's push endpoint.

Because a `liverevision` now exists, `currentrevision` carries only the fields that differ from it. `terms` is included with the updated `countries` array (array fields replace in full, not merge). `inventory` is included with an updated `contentcomp` carrying the new `excl` entry — `devicecomp` and `appcomp` are unchanged and omitted.

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

## Step 4: Seller Accepts the Revision

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

## Step 5: Buyer Polls Seller's Status Endpoint

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
