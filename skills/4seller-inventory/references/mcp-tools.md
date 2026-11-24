# MCP tools (4Seller inventory)

Verified against the live endpoint on 2026-11-24. The live schema and responses remain
authoritative; re-verify after server changes.

## `list_stores`

No parameters. Returns a bare array (no `total`/`records` envelope).

| Field | Meaning |
|---|---|
| `erpShopId` | Shop ID inside 4Seller |
| `shopName` | Seller-defined display name |
| `platform` / `platformName` | Platform code and display name |
| `site` | Site code; may be empty for single-site platforms |
| `authStatus` | `1` = authorized; any other value = authorization missing or expired |
| `supportPublish` | `true` = the shop can receive listing drafts |

## `list_inventory`

### Parameters

| Param | Type | Required | Default / limit | Note |
|---|---|---|---|---|
| `warehouseId` | integer | no | all warehouses | **Works**; verified to filter `total` |
| `availableMin` / `availableMax` | integer | no | — | **Works**; verified (`availableMin=1` dropped zero-stock rows) |
| `onHandMin` / `onHandMax` | integer | no | — | Quantity range on on-hand |
| `pageCurrent` | integer | no | `1` | 1-based |
| `pageSize` | integer | no | `20`, max `100` | Request ≥ 100 explicitly when scanning |
| `skuIdList` | array of integers | no | max 100 ids | ⚠ **Ignored by the server** — see below |

**Verified limitation:** passing `skuIdList` (integer or string ids) does **not** filter. The
response keeps the unfiltered `total` and returns unrelated rows. Filter by `skuId` client-side
after fetching, or narrow with `warehouseId` and quantity ranges. Report this upstream.

### Returns

```json
{
  "pageCurrent": 1,
  "pageSize": 100,
  "total": 5407,
  "records": [ { ... } ]
}
```

One record per SKU × warehouse × location:

| Field | Meaning | Pitfall |
|---|---|---|
| `sku` | Merchant SKU code | Lookup only — not a filter parameter |
| `skuId` | Internal numeric ID | Part of the record key; filter on it client-side |
| `productName`, `image` | Display data | — |
| `warehouseId`, `warehouseName` | Warehouse | IDs are tenant-specific |
| `locationId`, `locationName` | Stock location | Required for adjustments |
| `availableQty` | Sellable now | The only "can I sell" number |
| `onHandQty` | Physical on hand incl. committed | Never present as sellable |
| `reservedQty`, `transferReservedQty`, `lockedQty` | Already excluded from `availableQty` | Do not subtract again |
| `incomingQty`, `inTransferQty` | Expected inbound / in-flight transfer | Not stock yet |
| `safetyStockQty` | Configured floor | Absent ≠ 0 |

Rows repeat across pages: de-duplicate on `skuId + warehouseId + locationId` before aggregating,
and treat a conflicting duplicate as a data problem rather than an average.

## `list_inventory_flows`

### Parameters

| Param | Type | Required | Note |
|---|---|---|---|
| `startTime` | string | **yes** | `yyyy-MM-dd` or `yyyy-MM-dd HH:mm:ss` |
| `endTime` | string | **yes** | same format; keep each query ≤ 92 days |
| `warehouseId` | integer | no | **Works** (verified) |
| `businessType` | string | no | **Works** (verified) — see values below |
| `changeType` | string | no | **Works** (verified) — see values below |
| `referenceNumber` | string | no | Fuzzy match on the related document number |
| `pageCurrent` / `pageSize` | integer | no | `1` / `20`, max `100` |
| `skuIdList` | array | no | max 100 — ⚠ **also ignored** (verified) |

### Observed values (broader than the declared enum)

| Field | Values seen | Declared in schema (incomplete) |
|---|---|---|
| `businessType` | `sync_3pf_inventory`, `order`, `mcp_add_inventory_manually`, `mcp_adjust_inventory` | `add_inventory_manually`, `adjust_inventory`, `order` |
| `changeType` | `add`, `deduction`, `take_up` | `add`, `deduction` |

Treat these lists as examples, not a closed set, and read the live response before mapping.

### Returns

Same envelope (`pageCurrent`, `pageSize`, `total`, `records`). Each row:

| Field | Meaning |
|---|---|
| `flowId` | Movement ID |
| `businessType` / `businessTypeName` | Machine code and display name of the action |
| `changeType` | `add` / `deduction` / `take_up` |
| `oldAvailableQty` / `newAvailableQty` | Available before / after |
| `oldOnHandQty` / `newOnHandQty` | On-hand before / after |
| `onHandChangeQty` | Signed difference applied |
| `oldAverageCost` / `newAverageCost` | Cost basis before / after |
| `referenceNumber` | Related order or document number |
| `sku`, `skuId`, `productName`, `image` | Product identity |
| `warehouseId`, `warehouseName`, `locationId`, `locationName` | Where it happened |
| `createTime` | ISO-8601 with UTC offset (input uses date-only format) |

## `adjust_inventory`

### Parameters

| Param | Type | Required | Note |
|---|---|---|---|
| `skuId` | integer | **yes** | Internal SKU ID |
| `warehouseId` | integer | **yes** | Warehouse |
| `locationId` | integer | **yes** | From `list_inventory`; required when a SKU spans locations |
| `mode` | string | **yes** | `increase` or `set` |
| `qty` | integer | **yes** | `increase` needs `> 0`; `set` needs `>= 0` |
| `expectedCurrent` | integer | no (strongly recommended) | Current `onHandQty`; a mismatch rejects the call, preventing overwrites |
| `safetyStock` | integer | no | Safety-stock value applied with the change |

### Behaviour

- One call changes one SKU record; run several SKUs separately.
- Composite/bundle SKUs are rejected.
- A successful write creates a `flow` record and may synchronize to sales platforms
  asynchronously — 4Seller success does not prove platform state.
- Response body not exercised in this verification (read-only checks only): read the live
  response before depending on its fields.

## Verified behaviours worth remembering

1. `skuIdList` is ignored on both inventory tools — filter client-side by `skuId`.
2. `pageSize` defaults to 20; always ask for 100 when scanning, and page until `page * pageSize >= total`.
3. Empty result = no record under the filters, never zero stock.
4. All values are a snapshot; platform back offices remain authoritative.
