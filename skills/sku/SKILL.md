---
name: sku
description: Check available and on-hand stock or review inventory movements in 4Seller ERP by SKU, warehouse, or stock location. Use for 4Seller stock queries and corrections. Adjustments support adding a positive quantity (increase) or setting on-hand stock to a specified quantity (set), after reading and confirming current stock.
license: MIT
compatibility: "Requires 4Seller MCP tools list_stores, list_inventory, list_inventory_flows, adjust_inventory."
---

# 4Seller SKU inventory

| # | Capability | Tool |
|---|---|---|
| 1 | **Current stock quantity** — how much is available and on hand for a SKU, in which warehouse and location | `list_inventory` |
| 2 | **Inventory flow query** — what moved in or out, when, by whom, with before/after quantities | `list_inventory_flows` |
| 3 | **Adjust inventory** — increase a quantity or set it to a target value | `adjust_inventory` |

Field names, meanings, parameters, verified limits and known server limitations:
[MCP tools](references/mcp-tools.md).

## Connection check

If the 4Seller tools are not available in this session, the MCP server is not connected yet. The
4Seller plugin includes the server configuration. Have the user check `/mcp` and complete
4Seller sign-in when prompted. For a skill-only install, ask the user to run
`claude mcp add --transport http 4seller-prod https://www.4seller.com/api/mcp` or
`codex mcp add 4seller-prod --url https://www.4seller.com/api/mcp` followed by
`codex mcp login 4seller-prod`. Do not try to reach 4Seller data by any other route.

## Step 0 — scope first

Call `list_stores` to confirm which shops the account can see, then name warehouses by
`warehouseId` and `warehouseName` from stock data. Never reuse warehouse IDs from examples —
they belong to another tenant.

## 1. Current stock quantity

- A merchant code `sku` is not the internal `skuId`. `skuIdList` is **ignored by the server**
  (verified), so resolve code → `skuId` with `list_inventory` and filter rows client-side.
- Narrow the scan with `warehouseId` or the quantity ranges; `pageSize` defaults to 20 and caps
  at 100, so ask for 100 explicitly.
- Paginate with `pageCurrent` / `pageSize` until `page * pageSize >= total`; de-duplicate on
  `skuId + warehouseId + locationId` when rows repeat across pages.
- Report `availableQty` when the question is "how many can I sell"; report `onHandQty`
  when it is "how many are physically there". Say which one you used.
- An empty result means **no record under these filters**, not zero stock.

## 2. Inventory flow query

- Always request `startTime` and `endTime`; one query spans at most **92 days**, so split
  longer ranges into chunks and drop duplicated boundary rows.
- Filter with `businessType` when the user asks about one kind of change. Manual edits appear
  as `mcp_add_inventory_manually` and `mcp_adjust_inventory` — keep them separate from trading
  movements rather than summing everything.
- Read the live schema for the exact time format and field names before building the query.

## 3. Adjust inventory

Read, confirm, write, verify — in that order:

1. Locate `skuId`, `warehouseId`, `locationId` with `list_inventory` and read the current
   `onHandQty`. If one SKU sits in several locations, execute only when the location is
   specified or unambiguous.
2. Present the plan: SKU, warehouse, location, mode (`increase` needs `qty > 0`; `set` needs
   `qty >= 0`), current value and expected new value. Ask for confirmation unless the user
   already authorized this exact change.
3. Call `adjust_inventory` with `expectedCurrent` set to the value just read. Composite/bundle
   SKUs cannot be adjusted — report that instead of forcing it. For several SKUs, run each one
   separately and report each result.
4. If the snapshot is rejected, re-read and re-check whether the change is still valid. If the
   call times out, query stock and flows **before** retrying: never add the same quantity twice.
5. Confirm afterwards with `list_inventory` and report before → after per location. A successful
   4Seller write only proves the 4Seller change; platform synchronization may be asynchronous.

## Accuracy rules

- Snapshot values, not live ones — always attach the collection time.
- Missing fields are data gaps, not zeros; skip a conclusion rather than inventing one.
- Never subtract `reservedQty`, `transferReservedQty` or `lockedQty` from `availableQty`
  again; they are already excluded.
- `incomingQty` and `inTransferQty` are not stock yet.

## Out of scope

Orders, shipping labels, carrier rates, profit, ads and platform messages have no tools here.
Cross-platform real-time stock stays authoritative in each marketplace back office. Stock writes
are limited to what `adjust_inventory` supports.
