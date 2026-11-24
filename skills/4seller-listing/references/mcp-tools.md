# MCP tools (4Seller listing)

Verified against the live endpoint on 2026-11-24. The live schema and responses remain
authoritative; re-verify after server changes.

## `list_stores`

No parameters; returns a bare array.

| Field | Meaning |
|---|---|
| `erpShopId` | Shop ID; the value passed as `shopIds` |
| `shopName` | Display name |
| `platform` / `platformName` | Platform code and display name |
| `site` | Site code; may be empty |
| `authStatus` | `1` = authorized |
| `supportPublish` | `true` = can receive listing drafts |

Only `authStatus = 1` and `supportPublish = true` are valid draft targets.

## `save_crawl_draft` — writes the crawl box record

### Required

| Param | Type | Note |
|---|---|---|
| `sourceUrl` | string | Source product URL; used for traceability and duplicate detection |
| `title` | string | Product title |

### Optional

| Param | Type | Note |
|---|---|---|
| `sourcePlatform` | string | Source platform code, e.g. `AMAZON` / `ALIEXPRESS` |
| `msku` | string | Merchant SKU |
| `categoryId`, `categoryName`, `categoryPath` | string | Original platform category and full path |
| `brand`, `originCountry` | string | Brand; origin country code, e.g. `US` / `CN` |
| `shortDescription` | string | Short description |
| `productDescription` | string | Long description; HTML is cleaned, content beyond 60000 characters is truncated |
| `bulletPoint` | array | Bullet points |
| `mainImage` | string | Main image; falls back to the first `imageUrlList` entry |
| `imageUrlList` | array | Gallery (main + additional images) |
| `videoUrl`, `videoPostUrl` | string | Video and its poster |
| `highPrice`, `lowPrice` | number | Original and discounted price; each falls back to the other, then to the variant max/min |
| `currency` | string | ISO 4217 code. **Mandatory whenever any price is present** — never defaulted |
| `inventory` | integer | Stock quantity; defaults to `1000` when absent |
| `packageWeight` | number | `weightUnit`; defaults to grams when omitted |
| `packageLength`, `packageWidth`, `packageHeight` | number | `dimensionUnit`; defaults to cm |
| `itemSpecifics` | string | `name:value` lines; 4Seller maps these into the draft |
| `identifierCode`, `identifierCodeType` | string | Barcode and its type, e.g. `UPC` / `EAN` / `ISBN` |
| `hasVariant` | boolean | Derived from a non-empty `variants` array when omitted |
| `attributes` | array | Merged from variant `variantAttributes` when omitted |
| `variants` | array | One entry per real combination — see below |
| `shopIds` | array | Target `erpShopId` values; including them starts draft generation |
| `site`, `siteList` | string / array | Target site(s); action-only, used when generating drafts |

### Variant entry

`variantName`, `msku`, `highPrice`, `lowPrice`, `currency`, `inventory`, `variantImage`,
`imageUrlList`, `packageWeight`, `packageLength`, `packageWidth`, `packageHeight`,
`identifierCode`, `identifierCodeType`, `variantAttributes`.

Fallbacks when omitted: `variantName` → product `title`; `currency` → product currency;
`imageUrlList` → `variantImage`.

### Behaviour

- Fields outside this list are not stored (e.g. `productAttributes`, `searchTerms`, `sku`).
- Each call creates a new crawl-box record; the same `sourceUrl` again returns `repeat: true`.
- `shopIds` / `site` / `siteList` are action parameters and are **not** stored on the record.
- With `shopIds`, store checks run first; an unauthorized shop or unsupported platform fails
  the whole call before the async job is queued.

### Returns

`crawlDraftId`, `repeat` boolean, and — in save-and-generate mode — `scheduleCode`.

## `create_listing_draft`

| Param | Type | Required | Note |
|---|---|---|---|
| `crawlDraftId` | integer | **yes** | From `save_crawl_draft` |
| `shopIds` | array | **yes** | Target `erpShopId` values |
| `site` | string | no | Single target site |
| `siteList` | array | no | Multiple target sites |

Behaviour: asynchronous — returns `scheduleCode` only; `draftIds` is empty. Not idempotent:
calling twice for the same `crawlDraftId` and shop creates a second draft. Draft pricing comes
from the store price rule, not from this call. Only drafts are created; nothing is published.

## `get_draft_status`

| Param | Type | Required |
|---|---|---|
| `scheduleCode` | string | **yes** |

Returns:

| Field | Meaning |
|---|---|
| `total` | Drafts attempted in this run |
| `succeed` | Drafts created |
| `failed` | Drafts that failed |
| `failedList` | `title` and `reason` per failure |

Finished when `succeed + failed == total`. Poll every few seconds; stop after a few minutes.
`failed > 0` is an outcome to report, not a tool error. An unknown or expired `scheduleCode`
means the run cannot be located.

## Verified behaviours worth remembering

1. A price without `currency` is rejected — read the currency from the source, never assume.
2. `repeat: true` is a duplicate signal, not an error: surface it and let the user decide.
3. Save-and-generate mode is not re-runnable per product; a retry creates a second draft.
4. 4Seller does not read the source page: if data is missing, the caller must fill it in.
