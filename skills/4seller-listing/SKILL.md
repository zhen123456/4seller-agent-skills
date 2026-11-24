---
name: 4seller-listing
description: Convert listing raw data — an Excel/CSV/XLSX export, one or many product URLs, pasted structured data, or platform API output — into listing drafts for target shops in 4Seller. Data is normalized, validated, written to the 4Seller crawl box, then listing drafts are generated for the selected shops and tracked to completion. Use when the user has a product list or spreadsheet to turn into drafts, wants to push one or many products to several shops or sites, or needs draft generation progress and failure reasons — via their authorized 4Seller account.
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

The crawl box is the single entry point: nothing reaches a shop before its record exists there.
**This skill never publishes to a marketplace** — publishing stays a user action inside 4Seller.

## Connection check

If the 4Seller tools are not available in this session, the MCP server is not connected yet. The
4Seller plugin registers it on install and asks the user to sign in (Claude Code: `/mcp`; Codex: the
OAuth prompt at install). For a skill-only install, ask the user to run
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

Bulk runs: the mapping table and the per-row result report are the deliverables, so a second
run on the same file produces the same records. Template and batch rules:
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

- Build one product object per row/URL. Never invent a value: a required field that the source
  does not provide is either asked for or left out **and reported**.
- `sourceUrl` is required by the tool. When a spreadsheet row has no URL, ask the user for a
  reference URL per product, or a batch rule they own; do not fabricate one.
- Every price — product or variant — must carry an ISO 4217 currency. `currency` is never defaulted.
- Variants must be real combinations with real attribute values; a flattened "one size/one color"
  row loses data and should be flagged rather than guessed.
- Record the mapping you used (source column → tool field) and the columns you could not map.

## Step 3 — validate before writing

Show a row-level preview table: product title, `sourceUrl`, price + currency, image count,
variant count, missing required fields, mapped/unmapped columns. Ask for confirmation. Rows that
fail validation are excluded from the write and reported — do not silently drop them.

## Step 4 — write to the crawl box

One `save_crawl_draft` call per product. Two modes:

| Mode | Call | Result |
|---|---|---|
| Save only | omit `shopIds` | crawl-box record + `crawlDraftId` |
| Save and generate | include `shopIds` | crawl-box record **and** drafts start generating, plus `scheduleCode` |

`shopIds`, `site` and `siteList` are action parameters only — they are not stored on the record.
Deduplicate before writing: the same `sourceUrl` already in the crawl box returns `repeat: true`;
surface it and let the user decide instead of duplicating silently.

## Step 5 — generate drafts (save-only mode)

Call `create_listing_draft` per product with its `crawlDraftId` and the confirmed `shopIds`.
Not idempotent: a second call for the same record and shop creates a second draft. Draft pricing
comes from the store's price rule, not from this call.

## Step 6 — track the batch

Collect every `scheduleCode` (from mode B or from Step 5) and poll `get_draft_status` until
`succeed + failed == total` per run. Report a matrix: product × shop × outcome, each failure with
its `reason`. `failed > 0` is a result to report, not a tool error. If a run is still going after a
few minutes, report confirmed progress and stop polling.

## Batch discipline

- Work in confirmed chunks (for example 20–25 products per preview) and keep a running result table.
- One bad row must not abort the batch: collect row-level errors and continue with valid rows.
- Unknown write outcome (timeout, interrupted call): re-check the crawl box (`repeat`) before retrying.
- Never re-run a save-and-generate batch as a whole; re-run only rows that have no record yet.

## Failure handling

| Situation | Action |
|---|---|
| Authorization invalid or expired | Ask the user to re-authorize, then retry once |
| Missing `sourceUrl` or `title` | Ask the user; do not fabricate |
| Price without currency | Read the currency from the source and retry, or drop the price |
| `repeat: true` | Surface to the user; continue only on their decision |
| Shop not authorized / platform unsupported | Deterministic: change or reconnect the shop, do not retry |
| Unknown or expired `scheduleCode` | Say the run cannot be located; do not guess |
| Rate limited | Back off, then retry once |
