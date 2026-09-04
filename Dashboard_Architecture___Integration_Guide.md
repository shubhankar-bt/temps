# Fincore Dashboard Architecture & Integration Guide

## 1. Executive Summary & Architectural Decision
The Fincore platform is introducing interactive, drill-down dashboards. To support this, we evaluated whether to build Druid query execution capabilities into the existing `Dashboard Service` or reuse the `Analytics Service`.

**Decision:** We will adopt the **Separation of Concerns (BFF Pattern)**. 
* The **Dashboard Service** will act as a layout manager (storing user preferences, widget positions, and chart types).
* The **Analytics Service** will act as the single source of truth for all data execution, semantic mapping, and Druid cluster interaction.

### Evaluation of Approaches (Pros & Cons)

#### ❌ Anti-Pattern: Duplicating Druid Execution in Dashboard Service
* **Pros:** Dashboard backend team feels autonomous; no cross-service API dependencies.
* **Cons:** * **Massive Tech Debt (Violates DRY):** We would have to duplicate the Druid SSL connection pool, Avatica drivers, and the In-Memory Mock Engine for ST testing.
  * **Logic Drift:** If a calculation for "Profitability" changes in the Semantic Layer, the Dashboard Service will show different numbers than the Grid Reports.
  * **Security Risk:** RBAC rules would need to be maintained in two places.

#### ✅ Recommended Architecture: Unified Semantic Layer
* **Pros:** * **Zero Logic Duplication:** The Analytics Engine doesn't care if the UI renders a Data Grid or a Bar Chart. It just returns JSON.
  * **Instant ST Mocking:** The Dashboard automatically works in System Testing environments via the existing Java Mock Engine.
  * **Highly Scalable:** The Analytics Service is already optimized with Virtual Threads and Druid Bitmap indexes.
* **Cons:** Requires the UI to fetch layouts from one service and data from another. (Standard Micro-Frontend pattern).

---

## 2. Target Architectural Flow
The Frontend (React/Angular) orchestrates the flow. It uses the Dashboard Service to build the UI skeleton, then hydrates the charts using the Analytics Service.

1. **UI Load:** Frontend calls `GET /api/dashboards/{userId}` on the **Dashboard Service**.
2. **Skeleton Render:** Dashboard Service returns the layout: *"Render a Bar Chart here for 'Current A/c', 'Saving A/c', 'Cash Credit'"*.
3. **Data Hydration:** Frontend fires parallel asynchronous POST requests to `POST /api/analytics/drill-down` on the **Analytics Service** to fetch the data for those charts.
4. **Drill-Down:** When a user clicks a bar on the chart, the UI fires a new request to the Analytics Service for the next hierarchy level.

---

## 3. Frontend Integration Guide (API Contracts)

A chart is simply a visual representation of a data grid. To build a Bar Chart comparing different deposit types, you map logical HeadCodes into "Buckets" and fetch their totals.

### Scenario A: Rendering the Top-Level Chart
Assume the Dashboard needs to render a Bar Chart comparing "Current A/c" (Headcodes 01042, 01043, 02723...) vs "Saving Bank A/c" (01036). 

**The UI fires parallel requests to the Analytics Service for each "Bar".**

**Payload for "Current A/c" Bar:**
*We request grouping by `ReportDate` because we want all these headcodes summed up into one single total number for the chart.*
```json
{
  "viewCode": "YSA_BANK",
  "headCodes": ["01042", "01043", "02723", "01044", "01045", "01046"], 
  "dimensions": ["ReportDate"], 
  "metrics": ["AMOUNT"],
  "filters": [
    {
      "logicalField": "ReportDate",
      "operator": "EQUALS",
      "value": "12-04-2026"
    }
  ]
}
```

**Response from Analytics Service:**
```json
{
  "meta": { "status": "SUCCESS", "level": "ReportDate", "rowCount": 1, "executionTimeMs": 45 },
  "data": [
    {
      "id": "reportdate_12_04_2026",
      "name": "12-04-2026",
      "AMOUNT": 1500000000.00,  // <-- Map this number to the 'Current A/c' Bar Height!
      "hasChildren": true
    }
  ]
}
```

### Scenario B: Drill-Down (Clicking the Chart)
The user clicks the "Current A/c" bar to see its Geographical breakdown. The UI transitions the Bar Chart into a Pie Chart (or drilled Bar Chart) showing Domestic vs. Foreign.

**Payload for Drill-Down:**
*Notice how the `headCodes` array remains exactly the same. We just advance the `dimensions` array.*
```json
{
  "viewCode": "YSA_BANK",
  "headCodes": ["01042", "01043", "02723", "01044", "01045", "01046"], 
  "dimensions": ["Geography"], // <-- Advance hierarchy
  "metrics": ["AMOUNT"],
  "filters": [
    {
      "logicalField": "ReportDate",
      "operator": "EQUALS",
      "value": "12-04-2026"
    }
  ]
}
```

**Response from Analytics Service:**
```json
{
  "meta": { "status": "SUCCESS", "level": "Geography", "rowCount": 2, "executionTimeMs": 38 },
  "data": [
    {
      "id": "geography_domestic",
      "name": "Domestic",
      "AMOUNT": 1200000000.00, // <-- Render as Pie Chart Slice 1
      "hasChildren": true
    },
    {
      "id": "geography_foreign",
      "name": "Foreign",
      "AMOUNT": 300000000.00, // <-- Render as Pie Chart Slice 2
      "hasChildren": true
    }
  ]
}
```

---

## 4. Best Practices for High Throughput

1. **Parallel Execution (Frontend):** When loading a dashboard with 4 widgets, the UI should use `Promise.all()` to fire the 4 `POST /api/analytics/drill-down` requests concurrently. The Java Virtual Threads and Druid cluster are designed for high concurrency and will resolve them in milliseconds.
2. **Browser Connection Limits:** Browsers limit concurrent connections to the same domain (usually 6). If a dashboard has >6 charts, consider having the UI stagger the requests (load top-fold charts first, then bottom-fold). 
3. **Caching:** The Analytics Service has a Redis `@Cacheable` layer on the `drill-down` endpoint based on the payload hash. If 100 users load the default "Bank Level Dashboard" at 9:00 AM, Druid is only queried once. The other 99 requests are served from Redis in <5ms.
4. **Zero-Data State:** If a chart drill-down returns `0` rows (or rows with `AMOUNT: 0`), the UI chart library must gracefully render a "No Data / $0" state rather than breaking the UI. The Analytics service is configured to safely pad zero-values where applicable.