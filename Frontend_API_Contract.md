# API Contract: Multi-HeadCode Drill-Down Implementation

## 1. /api/analytics/config (On Application Load)
The hierarchy array has been updated from the backend metadata. `HeadCode` is now the root dimension (Level 1). You must start your drill-down sequence with `"HeadCode"`.

**Response Excerpt:**
```json
{
  "reportCode": "YSA",
  "displayName": "Yield & Spread Analysis",
  "views": [
    {
      "viewCode": "YSA_BANK",
      "displayName": "Bank Level",
      "hierarchy": [
        "HeadCode",    // <-- NEW: This is now the first element!
        "Geography", 
        "Circle", 
        "CGL", 
        "Product", 
        "Branch-Circle"
      ],
      "availableMetrics": [
        { "logicalName": "AMOUNT", "displayName": "Total Amount" }
      ]
    }
  ]
}
```

---

## 2. /api/analytics/drill-down (Level 1: Grid Load)
Triggered when the user selects Head Codes in the dialog and clicks Search. 
**Rule:** Send the selected head codes (max 5) in the new root-level `headCodes` array. Request `HeadCode` as the first dimension.

**Exact UI Request Payload:**
```json
{
  "viewCode": "YSA_BANK",
  "headCodes": ["10001", "10002", "10003"], 
  "dimensions": ["HeadCode"],
  "metrics": ["AMOUNT"],
  "filters": []
}
```

**Exact Backend Response:**
```json
{
  "meta": {
    "status": "SUCCESS",
    "level": "HeadCode",
    "rowCount": 3,
    "executionTimeMs": 42
  },
  "data": [
    {
      "id": "headcode_10001",
      "name": "10001",
      "AMOUNT": 5000000.00,
      "hasChildren": true
    },
    {
      "id": "headcode_10002",
      "name": "10002",
      "AMOUNT": 3500000.00,
      "hasChildren": true
    },
    {
      "id": "headcode_10003",
      "name": "10003",
      "AMOUNT": 1250000.00,
      "hasChildren": true
    }
  ]
}
```

---

## 3. /api/analytics/drill-down (Level 2: User Clicks a Row)
Triggered when the user clicks the row for Head Code **"10001"** to see its geographical breakdown.
**Rule:** Leave the global `headCodes` array exactly as it was. Advance the `dimensions` array to `"Geography"`. Push the clicked row into the `filters` array.

**Exact UI Request Payload:**
```json
{
  "viewCode": "YSA_BANK",
  "headCodes": ["10001", "10002", "10003"], 
  "dimensions": ["Geography"],
  "metrics": ["AMOUNT"],
  "filters": [
    {
      "logicalField": "HeadCode",
      "operator": "EQUALS",
      "value": "10001"
    }
  ]
}
```

**Exact Backend Response:**
```json
{
  "meta": {
    "status": "SUCCESS",
    "level": "Geography",
    "rowCount": 2,
    "executionTimeMs": 38
  },
  "data": [
    {
      "id": "geography_domestic",
      "name": "Domestic",
      "AMOUNT": 3000000.00,
      "hasChildren": true
    },
    {
      "id": "geography_foreign",
      "name": "Foreign",
      "AMOUNT": 2000000.00,
      "hasChildren": true
    }
  ]
}
```

### Note on Breadcrumbs and Navigation
If the user clicks "Back" on the breadcrumb from `Geography` back to `HeadCode`, simply revert to the **Level 1 Request Payload** (empty out the filters array and set dimensions back to `["HeadCode"]`).