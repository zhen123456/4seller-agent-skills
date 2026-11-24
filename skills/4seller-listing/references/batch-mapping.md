# Batch mapping (raw data → crawl box records)

Rules for turning a spreadsheet or a list of products into many crawl-box records without
losing data or duplicating writes.

## Reading the source

- Read every sheet and every row. Keep the raw cell values; normalize afterwards.
- Preserve identifiers exactly: SKU codes with leading zeros, ASIN/listing IDs, URLs.
- Treat header names as untrusted: the same field can be called `Product Title`, `NAME`,
  `item_name` or `title_local`. Map by meaning, not by position.
- Detect multi-row variants: repeated product identifier with differing variant columns means one
  product with several variants, not several products.
- Empty cells are missing data, never zero and never "false/true".

## Mapping table (deliverable, keep it with the run)

| Source column | Tool field | Rule |
|---|---|---|
| Product URL | `sourceUrl` | Required; must be real. Ask when absent |
| Title | `title` | Required |
| Price | `highPrice` / `lowPrice` | Send with `currency` |
| Currency | `currency` | ISO 4217; never defaulted |
| Images | `mainImage`, `imageUrlList` | First image → main |
| Description | `shortDescription`, `productDescription`, `bulletPoint` | Clean HTML before sending |
| Category | `categoryId`, `categoryName`, `categoryPath` | Keep the full path when present |
| Variant columns | `variants[]` | One entry per real combination |
| Weight/size | `packageWeight`, `packageLength/Width/Height`, units | Convert to the store's units |
| Specs | `itemSpecifics` | `name:value` lines |
| Identifiers | `identifierCode`, `identifierCodeType` | Keep type and value together |

Anything without a target field is **unmapped data**: list it in the report so the user can decide
whether to extend the source or drop it. Do not stuff it into `attributes` blindly.

## Pre-write validation (per row)

| Check | Fail condition |
|---|---|
| `sourceUrl` | Missing, malformed host, or duplicated within the batch |
| `title` | Empty |
| Price ↔ currency | Any price without a currency, or a non-ISO-4217 code |
| Variants | Variant present without attribute values, or product marked as variant while `variants` is empty |
| Images | No image at all |
| Required business data | Category or identifier missing when the target platform needs it |

Valid rows proceed; invalid rows are reported with the reason and excluded.

## Per-row result report (deliverable)

```text
product | sourceUrl | shops | crawlDraftId | repeat | scheduleCode | status | reason
```

Status values: `draft-created`, `generating`, `succeeded`, `failed(<reason>)`, `skipped(invalid)`,
`skipped(repeat-unconfirmed)`, `blocked(auth)`.

## Duplicate and retry rules

- Same `sourceUrl` in the batch → one record only; confirm with the user before writing a second.
- `repeat: true` from the server means the crawl box already holds that URL: report it, do not
  overwrite, do not create a parallel record unless the user asks.
- Interrupted or timed-out write → check the crawl box before retrying; never re-issue the whole
  batch.
- Save-and-generate mode is not re-runnable per product: a second call creates a second draft.
