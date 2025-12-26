# TaxSaver API Documentation

## 1. Overview

This API enables tax-harvesting calculations for Indian investors. Users can either:

- Upload **Zerodha Excel exports** (Tax P&L + Holdings), or
- Submit **structured trade/position objects** (JSON).

The response includes realized P&L summaries, LTCG exemption remaining, and recommended actions (tax-free gains and loss harvesting).

---

## 2. Base URL, Headers, and Auth

- **Base URL:** `https://tax-saver.in`
- **Content-Type:** `application/json`
- **Authentication:** API_KEYS

---

## 3. Endpoint: Excel Upload

### 3.1 `POST /v1/excel`

Uploads Zerodha **Tax P&L XLSX** and **Holdings XLSX**, and returns tax-harvesting calculations and recommendations.

#### Request Body (JSON)

| Field                   | Type     | Required | Default | Description                                                                                          |
| ----------------------- | -------- | -------: | ------- | ---------------------------------------------------------------------------------------------------- |
| `userId`                | string   |      Yes | -       | Unique user identifier used for response/persistence.                                                |
| `minimumTradeThreshold` | number   |       No | `100`   | Minimum estimated tax impact (INR) required for inclusion in recommendations.                        |
| `pnl_file`              | object   |      Yes | -       | Zerodha Tax P&L XLSX payload: `{ filename, content }` where `content` is base64 encoded XLSX bytes.  |
| `holdings_file`         | object   |      Yes | -       | Zerodha Holdings XLSX payload: `{ filename, content }` where `content` is base64 encoded XLSX bytes. |
| `exclusion_list`        | string[] |  Planned | `[]`    | Symbols/tickers to exclude from all calculations.                                                    |

#### File payload shape

```json
{
  "filename": "holdings.xlsx",
  "content": "<BASE64_ENCODED_XLSX_BYTES>"
}
```

#### Example request (cURL)

```bash
curl -X POST "tax-saver.in/v1/excel"   -H "Content-Type: application/json"   -d '{
    "userId": "HNG098",
    "minimumTradeThreshold": 500,
    "pnl_file": { "filename": "taxpnl.xlsx", "content": "<base64...>" },
    "holdings_file": { "filename": "holdings.xlsx", "content": "<base64...>" },
    "exclusion_list": ["INFY", "TCS"]
  }'
```

#### Response (200 OK)

High-level shape (actual fields may evolve):

```json
{
  "uuid": "HNG098",
  "brokerage": "ZERODHA_EXCEL",
  "calculation": {
    "summary": {
      "realized": { "ltcg": 0, "stcg": 0, "ltcl": 0, "stcl": 0 },
      "ltcg_exemption_limit": 125000,
      "ltcg_exemption_utilized": 0,
      "ltcg_exemption_remaining": 125000
    },
    "recommendations": {
      "tax_free_gains": [],
      "loss_harvest": []
    },
    "advisor": [],
    "freeCalculation": { "amount": 0 },
    "calculation_timestamp": 1735000000,
    "expiry": 1735600000
  }
}
```

#### Excel format expectations (Zerodha)

- **Holdings file:** Zerodha Holdings XLSX file.
- **Tax P&L file:** Zerodha Tax P&L XLSX file.

#### Error responses

- **400**: Invalid JSON / missing `userId` / missing file payload fields
- **405**: Unsupported method (POST only)

---

### 3.2 `POST /v1/excel/bulk`

Bulk-submit multiple Excel calculation jobs in one request. Each job is equivalent to `POST /excel`.

#### Request Body

```json
{
  "jobs": [
    {
      "userId": "HNG098",
      "minimumTradeThreshold": 500,
      "pnl_file": { "filename": "taxpnl.xlsx", "content": "<base64>" },
      "holdings_file": { "filename": "holdings.xlsx", "content": "<base64>" },
      "exclusion_list": ["INFY"]
    }
  ]
}
```

#### Response Body

```json
{
  "results": [
    {
      "userId": "HNG098",
      "status": "ok",
      "brokerage": "ZERODHA_EXCEL",
      "calculation": { "...": "..." }
    }
  ]
}
```

---

## 4. Endpoint: Trades JSON

### 4.1 `POST /v1/trades`

Submit structured portfolio and trade data (JSON) instead of Excel and receive the same calculation envelope as `POST /excel`.

#### Design principles

1. Broker-agnostic schema that can be extended later.
2. Accept either a **pre-aggregated realized summary** or **detailed closed trades** (server computes ST/LT using dates).
3. Always require **current positions** to generate harvesting recommendations.

#### Request Body (JSON)

| Field                   | Type                | Required | Default | Description                                                                                                     |
| ----------------------- | ------------------- | -------: | ------- | --------------------------------------------------------------------------------------------------------------- |
| `userId`                | string              |      Yes | -       | Unique user identifier used for response/persistence.                                                           |
| `asOfDate`              | string (YYYY-MM-DD) |       No | Today   | Valuation date for market prices and term calculations (if needed).                                             |
| `minimumTradeThreshold` | number              |       No | `100`   | Minimum estimated tax impact (INR) required for inclusion in recommendations.                                   |
| `exclusion_list`        | string[]            |       No | `[]`    | Symbols/tickers to exclude from all calculations.                                                               |
| `positions`             | `Position[]`        |      Yes | -       | Current holdings/positions used to compute unrealized gains/loss and recommendations.                           |
| `realized_summary`      | `RealizedSummary`   |       No | -       | Optional pre-aggregated realized values for the FY (ltcg/stcg/ltcl/stcl).                                       |
| `closed_trades`         | `ClosedTrade[]`     |       No | `[]`    | Optional detailed closed trades. If provided (and `realized_summary` absent), server computes realized summary. |
| `metadata`              | object              |       No | -       | Optional client metadata (broker, tags, upload source).                                                         |

#### `Position` schema

A position represents current holdings of a symbol.

```json
{
  "symbol": "INFY",
  "type": "equity",
  "quantity_total": 10,
  "quantity_long_term": 6,
  "avg_buy_price": 1500.5,
  "market_price": 1700.0,
  "exchange": "NSE"
}
```

#### `RealizedSummary` schema

Use this if you already have FY realized values.

```json
{
  "stcg": 12000.0,
  "ltcg": 80000.0,
  "stcl": 0.0,
  "ltcl": 15000.0,
  "currency": "INR",
  "financial_year": "2025-2026"
}
```

#### `ClosedTrade` schema (optional alternative to `realized_summary`)

Use this if you want the server to compute realized ST/LT classification from dates.

```json
{
  "symbol": "TCS",
  "type": "equity",
  "quantity": 5,
  "buy_date": "2024-01-10",
  "sell_date": "2025-08-15",
  "buy_price": 3200.0,
  "sell_price": 3900.0,
  "charges": 0.0
}
```

**Classification:** Server derives holding period from `buy_date` and `sell_date` to classify as short-term vs long-term (rules configurable; default assumption for listed equity: holding period >= 12 months is long-term).

#### Example request (cURL)

```bash
curl -X POST "https://tax-saver.in/v1/trades"   -H "Content-Type: application/json"   -d '{
    "userId": "HNG098",
    "asOfDate": "2025-12-26",
    "minimumTradeThreshold": 500,
    "exclusion_list": ["INFY"],
    "positions": [
      {
        "symbol": "TCS",
        "type": "equity",
        "quantity_total": 10,
        "quantity_long_term": 10,
        "avg_buy_price": 3200.0,
        "market_price": 3900.0,
        "exchange": "NSE"
      }
    ],
    "realized_summary": {
      "stcg": 12000,
      "ltcg": 80000,
      "stcl": 0,
      "ltcl": 15000,
      "currency": "INR"
    }
  }'
```

#### Response (200 OK)

Same response envelope as `POST /excel`; brokerage identifies the input type.

```json
{
  "uuid": "HNG098",
  "brokerage": "TRADES_JSON",
  "calculation": { "...": "same structure as /excel..." }
}
```

#### Validation and error handling (recommended)

- **400**: missing `userId`, missing `positions`, invalid dates, negative quantities/prices, malformed objects
- **422 (optional)**: schema validation errors with per-field details (if you introduce strict validation)

---

### 4.2 `POST /v1/trades/bulk`

Bulk-submit multiple trade/position jobs in one request. Each job is equivalent to `POST /trades`.

#### Request Body

```json
{
  "jobs": [
    {
      "userId": "HNG098",
      "minimumTradeThreshold": 500,
      "exclusion_list": ["INFY"],
      "positions": [
        {
          "symbol": "TCS",
          "type": "equity",
          "quantity_total": 10,
          "quantity_long_term": 10,
          "avg_buy_price": 3200,
          "market_price": 3900
        }
      ],
      "realized_summary": {
        "stcg": 12000,
        "ltcg": 80000,
        "stcl": 0,
        "ltcl": 15000,
        "currency": "INR"
      }
    }
  ]
}
```

#### Response Body

```json
{
  "results": [
    {
      "userId": "HNG098",
      "status": "ok",
      "brokerage": "TRADES_JSON",
      "calculation": { "...": "..." }
    }
  ]
}
```
