---
name: 4seller-listing
description: Turn a spreadsheet (Excel/CSV), product URLs, pasted product data, or platform API output into 4Seller listing drafts. Data is normalized, saved to the 4Seller crawl box, and drafts are generated for one or many authorized shops, with progress and failure reasons. Use for bulk listing draft import, multi-shop drafts, and draft troubleshooting. Publishing stays a user action in 4Seller.
license: MIT
compatibility: "Requires 4Seller MCP tools list_stores, save_crawl_draft, create_listing_draft, get_draft_status."
---

# 4Seller listing

## What it does

Turns listing raw data, in whatever form the user already has it, into listing drafts:

```text
listing raw data (Excel/CSV/XLSX, product URL(s), pasted structured data, platform API output)
   → normalize + validate (field mapping, currency, variants, required fields)
   → 4Seller crawl box records (one per product)
   → listing drafts for the selected shops
```

The crawl box is 4Seller's staging area for product data. Each product is saved there before
drafts are generated for the selected shops.
**This skill never publishes to a marketplace** — publishing stays a user action inside 4Seller.

## Connection check

If the 4Seller tools are not available in this session, the MCP server is not connected yet. The
4Seller plugin includes the server configuration. Have the user check `/mcp` and complete
4Seller sign-in when prompted. For a skill-only install, ask the user to run
`claude mcp add --transport http 4seller-prod https://www.4seller.com/api/mcp` or
`codex mcp add 4seller-prod --url https://www.4seller.com/api/mcp` followed by
`codex mcp login 4seller-prod`. Do not try to reach 4Seller data by any other route.

## Accepted sources

| Source | How to read it | Normalization focus |
|---|---|---|
| Excel / CSV / XLSX export | Read every row of every sheet; keep the raw values | Column → field mapping, units, currency codes, variant rows |
| One or many product URLs | Read each page's authoritative data (structured markup and embedded state before rendered text) | One record per URL; variants and gallery completeness |
| Pasted structured data (JSON/table) | Use the fields as given | Missing required fields, currency pairing |
| Platform API output | Use the provider's objects | Status/unit conversion, identifier codes |

For bulk runs, provide the mapping table and per-product result report. Reuse the field mapping
for the same file and check previous write results before resuming. See the template and
[batch mapping](references/batch-mapping.md).

## Target shops and platforms

Start with `list_stores`; keep only shops where `authStatus = 1` **and** `supportPublish = true`.
Observed examples include Shopify, AliExpress and Temu. **Amazon is out of scope for this skill** —
do not offer or imply Amazon draft generation.

## Step 1 — agree on scope

Confirm: how many products, which source, which shops and sites, and whether the user wants one
shop or a product × shop matrix. Never bind a shop by name similarity alone.

## Step 2 — normalize

Read [MCP tools](references/mcp-tools.md) for the exact input list before building anything.

- Group rows by product before creating records. Use the source's parent product identifier when
  one is available and combine its variant rows into one product object. Distinct products each
  get one object.
- `sourceUrl` is optional for intake and preview. Preserve a real URL when provided and leave it
  blank in the preview when absent. At save time, omit the field if the live tool schema accepts
  omission. If the schema or server requires a URL, report the affected products as blocked
  pending real URLs and proceed with the other confirmed products. Never fabricate URLs.
- Every price — product or variant — must carry an ISO 4217 currency. `currency` is never defaulted.
- Variants must be real combinations with real attribute values; a flattened "one size/one color"
  row loses data and should be flagged rather than guessed.
- Record the mapping you used (source column → tool field) and the columns you could not map.

## Step 3 — validate before writing

Show one preview row per product with its title, `sourceUrl` if available, price and currency,
image and variant counts, missing required fields, and mapped and unmapped columns. Ask for
confirmation. Report validation failures and exclude those products from the write.

## Step 4 — write to the crawl box

One `save_crawl_draft` call per product. Two modes:

| Mode | Call | Result |
|---|---|---|
| Save only | omit `shopIds` | crawl-box record + `crawlDraftId` |
| Save and generate | include `shopIds` | crawl-box record **and** drafts start generating, plus `scheduleCode` |

`shopIds`, `site` and `siteList` are action parameters only — they are not stored on the record.
Deduplicate within the batch using a real source URL when present or a stable source product
identifier when it is absent. A returned `repeat: true` can mean a duplicate record has already
been saved. Report the result and avoid generating another draft for that product.

## Step 5 — generate drafts (save-only mode)

Call `create_listing_draft` per product with its `crawlDraftId` and the confirmed `shopIds`.
Not idempotent: a second call for the same record and shop creates a second draft. Draft pricing
comes from the store's price rule, not from this call.

## Step 6 — track the batch

Collect every `scheduleCode` (from save-and-generate mode or Step 5) and poll `get_draft_status` until
`succeed + failed == total` per run. Report a matrix: product × shop × outcome, each failure with
its `reason`. `failed > 0` is a result to report, not a tool error. If a run is still going after a
few minutes, report confirmed progress and stop polling.

## Batch discipline

- Work in confirmed chunks (for example 20–25 products per preview) and keep a running result table.
- Report errors for each affected product and continue with other valid products.
- Unknown write outcome (timeout or interrupted call): check the crawl box or draft status
  through an available read-only route. A second save is a write. If you cannot verify the
  outcome, report it as unknown and pause that product.
- Never re-run a save-and-generate batch as a whole; re-run only rows that have no record yet.

## Failure handling

| Situation | Action |
|---|---|
| Authorization invalid or expired | Ask the user to re-authorize, then retry once |
| Missing title | Ask for the missing title |
| Connected tool requires a URL | Ask for a real URL for the affected product and continue with the rest |
| Price without currency | Read the currency from the source and retry, or drop the price |
| `repeat: true` | Report what has already been saved or queued. Confirm before any further write for that product |
| Shop not authorized / platform unsupported | Deterministic: change or reconnect the shop, do not retry |
| Unknown or expired `scheduleCode` | Say the run cannot be located; do not guess |
| Rate limited | Back off, then retry once |
