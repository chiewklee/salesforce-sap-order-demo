# SAP Order System API — Endpoint Reference

The **only** app that talks to SAP. It encapsulates the core order flow — validate a sales
order (BAPI `TESTRUN`), then create + commit it (BAPI create + `BAPI_TRANSACTION_COMMIT`) —
behind a clean REST contract, so callers never see JCo/BAPI details.

## Base URL

| Environment | Base URL |
|---|---|
| Local (Mule EE standalone) | `http://localhost:8091` |
| CloudHub 2.0 | `https://sap-order-system-api.<region>.cloudhub.io` |

Content type for requests/responses: `application/json`.

## Error model (HTTP status = outcome)

| HTTP | Meaning | When |
|---|---|---|
| `200` | OK | `GET` endpoints |
| `201` | Created & committed | order passed validation and was created in SAP |
| `422` | Rejected | SAP business validation failed — **nothing created** |
| `503` | SAP unavailable (retryable) | SAP connectivity/timeout — caller may retry |

The `422` (business rejection) vs `503` (SAP down) split lets a caller's circuit breaker
distinguish "don't retry, the order is bad" from "retry, SAP is unreachable".

---

## 1. `GET /ping` — connectivity check

Verifies the SAP connection via `RFC_PING`.

```bash
curl -s http://localhost:8091/ping
```

```json
{
  "app": "sap-order-system-api",
  "status": "OK",
  "message": "SAP ECC connection verified via RFC_PING",
  "host": "your-sap-host.example.com",
  "client": "800"
}
```

---

## 2. `GET /api/salesorders/{id}` — read an order's status

Reads header + item status via `BAPI_SALESORDER_GETSTATUS`. `{id}` is the SAP
sales-document number; zero-padding to 10 chars is handled for you (`126063` and
`0000126063` both work).

```bash
curl -s http://localhost:8091/api/salesorders/126063
```

```json
{
  "salesDocument": "126063",
  "mechanism": "BAPI (synchronous) — BAPI_SALESORDER_GETSTATUS",
  "statusInfo": [
    {
      "DOC_NUMBER": "0000126063",
      "PURCH_NO_C": "SF-00000204",
      "ITM_NUMBER": "000010",
      "MATERIAL": "B-7000",
      "SHORT_TEXT": "Brochure: New high tech range",
      "REQ_QTY": "10.000",
      "SALES_UNIT": "ST",
      "DLV_STAT_I": "A"
    }
    // …one entry per item
  ],
  "messages": []
}
```

> `SHORT_TEXT` here is the **material-master** description, not the sales-order line
> text — see the note on `customerMaterial` below.

---

## 3. `POST /api/salesorders` — validate-then-create

Two steps in one synchronous call:

1. **Validate** — `BAPI_SALESORDER_CREATEFROMDAT2` with `TESTRUN='X'` (all SAP checks,
   no database change). Any `E`/`A` message stops here → `422`.
2. **Submit** — real `BAPI_SALESORDER_CREATEFROMDAT2` + `BAPI_TRANSACTION_COMMIT` in one
   transactional session (the LUW that persists the order) → `201`.

**All fields are optional** — demo-safe defaults are applied so a bare `{}` succeeds.

### Minimal request (everything defaulted → one B-7000 line)

```bash
curl -s -X POST http://localhost:8091/api/salesorders \
  -H "Content-Type: application/json" -d '{}'
```

### Realistic request (PO + line items — what the Order Intake API sends)

```bash
curl -s -X POST http://localhost:8091/api/salesorders \
  -H "Content-Type: application/json" -d '{
    "poNumber": "SF-DEMO-001",
    "items": [
      { "material": "B-7000", "quantity": "10", "unit": "ST", "plant": "1000",
        "customerMaterial": "SKU-1001 Sample Product A" },
      { "material": "B-7000", "quantity": "3",  "unit": "ST", "plant": "1000",
        "customerMaterial": "SKU-2002 Sample Product B" }
    ]
  }'
```

### Request fields (all optional; defaults shown)

| Field | Default | Notes |
|---|---|---|
| `docType` | `TA` | Standard order. `OR` does not exist in this demo system. |
| `salesOrg` | `1000` | |
| `distrChannel` | `10` | |
| `division` | `00` | |
| `poNumber` | `DEMO-PO-001` | Customer PO (header `PURCH_NO_C`). |
| `docDate` | `19980605` | Pinned to FY1998 (the demo system's CO plan version). |
| `priceDate` | `19980605` | |
| `reqDeliveryDate` | `19980609` | |
| `soldTo` | `0000099999` | Sold-to partner (role `AG`). |
| `items[]` | one `B-7000` line | See below. Empty → one default line. |

Per item (`items[]`):

| Field | Default | Maps to (SAP) |
|---|---|---|
| `itemNo` | `000010`, `000020`, … | `ITM_NUMBER` |
| `material` | `B-7000` | `MATERIAL` |
| `plant` | `1000` | `PLANT` |
| `quantity` | `1` | `TARGET_QTY` / schedule `REQ_QTY` |
| `unit` | `ST` | `TARGET_QU` |
| `reqDate` | order `reqDeliveryDate` | schedule `REQ_DATE` |
| `customerMaterial` | *(empty)* | `CUST_MAT35` — shows in VA03's **"Customer Material Numb"** column (KDMAT) |

> **Why `customerMaterial`?** SAP re-derives the item **Description** (`ARKTX`) from the
> material master, so `SHORT_TEXT` cannot override it for a material line. `CUST_MAT35` is a
> pure passthrough, so it's the reliable way to carry the caller's product identity into SAP.

### Response — `201 Created`

```json
{
  "validated": true,
  "submitted": true,
  "salesDocument": "0000126063",
  "mechanism": "Validate: …(TESTRUN='X') -> Submit: … + BAPI_TRANSACTION_COMMIT",
  "validationMessages": [ /* SAP RETURN rows */ ],
  "submissionMessages":  [ /* SAP RETURN rows */ ]
}
```

### Response — `422 Rejected` (nothing created)

```json
{
  "validated": false,
  "submitted": false,
  "salesDocument": "",
  "mechanism": "Validate: …(TESTRUN='X') — rejected before submit",
  "validationMessages": [
    { "messageType": "E", "id": "V1", "number": "382",
      "message": "Material ZZ-INVALID is not defined for sales org.1000, distr.chan.10, language DE" }
  ],
  "submissionMessages": []
}
```

### Response — `503 SAP unavailable`

```json
{ "error": "SAP_UNAVAILABLE", "errorType": "…", "message": "…", "retryable": true }
```

---

## Demo notes

- **A real `POST` creates an actual SAP order.** To demo the `422` path without
  committing anything, send an unknown material, e.g. `"material": "ZZ-INVALID"`.
- SAP validates the order **as a unit** — one bad line rejects the whole order (no
  partial creation).
- The demo system only has material **`B-7000`** confirmed valid for sales area
  `1000/10/00` + plant `1000`; that's why the Order Intake API maps every Salesforce
  line to `B-7000`.
- First RFC call after an idle period can fail with a CPIC "connection broken" / broken
  pipe at gateway `:3300` (stale pooled JCo connection) — a retry succeeds.

---

## Appendix A — Request → BAPI field mapping

What `POST /api/salesorders` actually sends to SAP. The **validate** step
(`TESTRUN='X'`) and the **submit** step build the *identical* payload — validate just
adds `import.TESTRUN = "X"`. On a clean submit a second RFC, `BAPI_TRANSACTION_COMMIT`
(`import.WAIT = "X"`), persists the LUW.

Source of truth: `src/main/mule/sap-order-system-api.xml` — validate builder ≈ lines
167-194, submit ≈ 247-263, commit ≈ 294. Defaults are applied in the "Resolve order
input" transform (≈ lines 127-160) *before* the BAPI payload is built.

### BAPI: `BAPI_SALESORDER_CREATEFROMDAT2`

**`import.ORDER_HEADER_IN`** (one header)

| Request field | BAPI field | Default | Notes |
|---|---|---|---|
| `docType` | `DOC_TYPE` | `TA` | Standard order. `OR` does not exist in this demo system. |
| `salesOrg` | `SALES_ORG` | `1000` | |
| `distrChannel` | `DISTR_CHAN` | `10` | |
| `division` | `DIVISION` | `00` | |
| `poNumber` | `PURCH_NO_C` | `DEMO-PO-001` | Customer PO; the Order Intake API sends `SF-<OrderNumber>` for traceability. |
| `docDate` | `DOC_DATE` | `19980605` | Pinned to FY1998 (CO plan version 0). |
| `priceDate` | `PRICE_DATE` | `19980605` | |
| `reqDeliveryDate` | `REQ_DATE_H` | `19980609` | Header requested delivery date. |
| *(fixed)* | `import.TESTRUN` | `X` on validate only | Omitted on submit. |

**`tables.ORDER_PARTNERS`** (one row)

| Request field | BAPI field | Default | Notes |
|---|---|---|---|
| *(fixed)* | `PARTN_ROLE` | `AG` | Sold-to party. |
| `soldTo` | `PARTN_NUMB` | `0000099999` | 10-char partner number. |

**`tables.ORDER_ITEMS_IN`** (one row per `items[]` entry)

| Request field | BAPI field | Default | Notes |
|---|---|---|---|
| `items[].itemNo` | `ITM_NUMBER` | `000010`, `000020`, … | Auto-numbered by position if omitted. |
| `items[].material` | `MATERIAL` | `B-7000` | Only `B-7000` is confirmed valid for `1000/10/00` + plant `1000`. |
| `items[].plant` | `PLANT` | `1000` | |
| `items[].quantity` | `TARGET_QTY` | `1` | |
| `items[].unit` | `TARGET_QU` | `ST` | |
| `items[].customerMaterial` | `CUST_MAT35` | *(empty)* | **Conditional** — the row omits `CUST_MAT35` entirely when blank. Capped at 35 chars. Shows in VA03's "Customer Material Numb" column (KDMAT); the reliable identity carrier since `ARKTX`/`SHORT_TEXT` can't be overridden for a material line. |

**`tables.ORDER_SCHEDULES_IN`** (one row per `items[]` entry)

| Request field | BAPI field | Default | Notes |
|---|---|---|---|
| `items[].itemNo` | `ITM_NUMBER` | matches the item row | Links the schedule line to its item. |
| `items[].reqDate` | `REQ_DATE` | order `reqDeliveryDate` → `19980609` | SAP auto-reschedules to a real future delivery date on save. |
| `items[].quantity` | `REQ_QTY` | `1` | Requested schedule-line quantity. |

### BAPI: `BAPI_TRANSACTION_COMMIT` (submit only, after a clean create)

| Field | BAPI field | Value | Notes |
|---|---|---|---|
| *(fixed)* | `import.WAIT` | `X` | Synchronous commit — waits for the update task, so the returned `salesDocument` is guaranteed persisted. |

### Fields SAP derives (not sent)

- **Item category** (`TAN`) — derived from `DOC_TYPE=TA` + the material's item-category group (`NORM`).
- **Item description** (`ARKTX`) — re-derived from the material master; caller input is ignored (hence `CUST_MAT35` for identity).
- **Confirmed delivery date** — rescheduled by ATP/availability check on save.
- **Pricing** (`NETWR`, conditions) — determined by SAP pricing procedure at `PRICE_DATE`.

---

## Appendix B — Sample payloads

### B.1  REST request — `POST /api/salesorders` (JSON)

What the caller (e.g. the Order Intake API) sends. Every field is optional — a bare `{}`
still creates one defaulted `B-7000` line.

```json
{
  "poNumber": "SF-00000210",
  "docType": "TA",
  "salesOrg": "1000",
  "distrChannel": "10",
  "division": "00",
  "soldTo": "0000099999",
  "docDate": "19980605",
  "priceDate": "19980605",
  "reqDeliveryDate": "19980609",
  "items": [
    { "itemNo": "000010", "material": "B-7000", "plant": "1000",
      "quantity": "10", "unit": "ST", "customerMaterial": "SKU-1001 Sample Product A" },
    { "itemNo": "000020", "material": "B-7000", "plant": "1000",
      "quantity": "3",  "unit": "ST", "customerMaterial": "SKU-2002 Sample Product B" }
  ]
}
```

### B.2  BAPI payload sent to SAP — `BAPI_SALESORDER_CREATEFROMDAT2` (XML)

The validate call is identical but adds `<TESTRUN>X</TESTRUN>` under `<import>`.

```xml
<BAPI_SALESORDER_CREATEFROMDAT2>
  <import>
    <ORDER_HEADER_IN>
      <DOC_TYPE>TA</DOC_TYPE>
      <SALES_ORG>1000</SALES_ORG>
      <DISTR_CHAN>10</DISTR_CHAN>
      <DIVISION>00</DIVISION>
      <PURCH_NO_C>SF-00000210</PURCH_NO_C>
      <DOC_DATE>19980605</DOC_DATE>
      <PRICE_DATE>19980605</PRICE_DATE>
      <REQ_DATE_H>19980609</REQ_DATE_H>
    </ORDER_HEADER_IN>
    <!-- <TESTRUN>X</TESTRUN>  present on the validate call only -->
  </import>
  <tables>
    <ORDER_PARTNERS>
      <row><PARTN_ROLE>AG</PARTN_ROLE><PARTN_NUMB>0000099999</PARTN_NUMB></row>
    </ORDER_PARTNERS>
    <ORDER_ITEMS_IN>
      <row><ITM_NUMBER>000010</ITM_NUMBER><MATERIAL>B-7000</MATERIAL><PLANT>1000</PLANT>
           <TARGET_QTY>10</TARGET_QTY><TARGET_QU>ST</TARGET_QU>
           <CUST_MAT35>SKU-1001 Sample Product A</CUST_MAT35></row>
      <row><ITM_NUMBER>000020</ITM_NUMBER><MATERIAL>B-7000</MATERIAL><PLANT>1000</PLANT>
           <TARGET_QTY>3</TARGET_QTY><TARGET_QU>ST</TARGET_QU>
           <CUST_MAT35>SKU-2002 Sample Product B</CUST_MAT35></row>
    </ORDER_ITEMS_IN>
    <ORDER_SCHEDULES_IN>
      <row><ITM_NUMBER>000010</ITM_NUMBER><REQ_DATE>19980609</REQ_DATE><REQ_QTY>10</REQ_QTY></row>
      <row><ITM_NUMBER>000020</ITM_NUMBER><REQ_DATE>19980609</REQ_DATE><REQ_QTY>3</REQ_QTY></row>
    </ORDER_SCHEDULES_IN>
  </tables>
</BAPI_SALESORDER_CREATEFROMDAT2>
```

### B.3  Commit — fired after a clean create (submit only)

```xml
<BAPI_TRANSACTION_COMMIT>
  <import><WAIT>X</WAIT></import>
</BAPI_TRANSACTION_COMMIT>
```

> The JSON `customerMaterial` is the only carrier of real product identity into SAP
> (`CUST_MAT35` → VA03 "Customer Material Numb"); the row **omits it entirely when
> blank**. `MATERIAL` is forced to `B-7000` — the only material valid for this demo's
> sales area.
