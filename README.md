# 4Seller Agent Skills

Use an AI assistant to check and adjust 4Seller inventory or prepare product listing drafts
for your authorized shops. Both skills require a 4Seller account and an authenticated connection
to the 4Seller MCP server, which lets your assistant use 4Seller tools.

Plugin installs include the MCP server configuration. Sign in to 4Seller to use it.
With the skills CLI or a manual copy, configure the server separately before signing in.

## Included skills

| Skill | Capabilities | Writes? | Try asking |
|---|---|---|---|
| [`4seller-inventory`](skills/4seller-inventory/SKILL.md) | Check available and on-hand stock, review stock movements, add units or set a target quantity | Reads current stock before making an authorized adjustment | “Show available and on-hand stock for SKU ABC-123 by warehouse.” |
| [`4seller-listing`](skills/4seller-listing/SKILL.md) | Turn a spreadsheet, product URLs, pasted data, or API output into listing drafts | Saves products to the crawl box (4Seller’s staging area for product data) and creates drafts after confirmation | “Check this spreadsheet for my Shopify shop and show me a draft preview before saving.” |

Neither skill publishes to a marketplace; publishing stays a user action inside 4Seller.

## Install

### Claude Code plugin (skills + MCP connection)

```text
/plugin marketplace add zhen123456/4seller-agent-skills
/plugin install 4seller@4seller-agent-skills
```

The plugin declares the server in `.mcp.json`; finish with the sign-in at `/mcp`.
The plugin exposes the skills as `/4seller:4seller-inventory` and `/4seller:4seller-listing`.

### Codex plugin (skills + MCP connection, auth at install)

```bash
codex plugin marketplace add zhen123456/4seller-agent-skills
codex plugin marketplace list          # note the registered marketplace name
codex plugin add 4seller@<marketplace-name-from-the-list>
```

Complete 4Seller sign-in when prompted. The marketplace requests authentication at installation.
Alternatively, open the repository in Codex and run `/plugins` to install from its repo marketplace,
and pin a version with `codex plugin marketplace add zhen123456/4seller-agent-skills --ref v0.1.0`.
Start a new Codex session, use `/mcp` to check the connection, and try the read-only SKU example
below.

### skills CLI (any supported agent)

```bash
npx skills add zhen123456/4seller-agent-skills
```

This installs skills only — connect the MCP server yourself:

```bash
claude mcp add --transport http 4seller-prod https://www.4seller.com/api/mcp   # Claude Code
codex mcp add 4seller-prod --url https://www.4seller.com/api/mcp               # Codex CLI
codex mcp login 4seller-prod
```

### Manual copy

| Client | Project scope | User scope |
|---|---|---|
| Claude Code | `.claude/skills/<skill>/` | `~/.claude/skills/<skill>/` |
| Codex | `.agents/skills/<skill>/` | `~/.agents/skills/<skill>/` (`~/.codex/skills/` also works) |

Keep `references/` next to `SKILL.md`.
Use the MCP setup commands above for your client and complete 4Seller sign-in.

## Verify

1. **Check inventory.** Start a new session and ask “Show available and on-hand stock for SKU
   ABC-123 by warehouse.” Replace `ABC-123` with a SKU in your 4Seller account.
2. **Preview a listing batch.** Provide a spreadsheet with a title and at least one image URL per
   product, plus an authorized target shop that supports listing drafts. If a row or variant has
   a price, include its ISO 4217 currency code. The source product URL is optional for the
   preview. If the connected MCP tool requires it to save a product, add a real URL for that
   product before saving. Ask to review the preview before any records are written.

## One install channel per machine

Identity here is name-based (no opaque IDs): marketplace `4seller-agent-skills`, plugin `4seller`,
skills `4seller-inventory` / `4seller-listing`, MCP server `4seller-prod`.

| Situation | What happens |
|---|---|
| Re-installing the same plugin | Replaces it — one slot per marketplace + plugin name |
| Re-running `npx skills add` | Updates the installed skill directories |
| Plugin install **plus** a manual copy | Duplicate copies in the plugin and skills directories |
| The same repository added as two marketplaces | Two markets, two same-named plugins — remove one |

Pick one route per machine and stay on it. To inspect or clean up:

```text
/plugin                                          # Claude Code: installed plugins
/plugin uninstall 4seller@4seller-agent-skills
codex plugin list                                # Codex
codex plugin remove 4seller@4seller-agent-skills
npx skills list                                  # skills CLI
npx skills remove 4seller-inventory 4seller-listing
```

If you installed an older `listing` or `sku` folder using the skills CLI or a manual copy,
remove that old copy after upgrading.

## Repository layout

```text
4seller-agent-skills/
├── .claude-plugin/
│   ├── marketplace.json      # Claude Code marketplace
│   └── plugin.json           # Claude plugin manifest
├── .codex-plugin/
│   └── plugin.json           # Codex plugin manifest
├── .agents/
│   └── plugins/marketplace.json   # Codex repo marketplace (auth on install)
├── .mcp.json                 # 4Seller MCP server, bundled with both plugins
├── skills/
│   ├── 4seller-inventory/{SKILL.md, references/mcp-tools.md}
│   └── 4seller-listing/{SKILL.md, references/mcp-tools.md, batch-mapping.md}
├── README.md
└── LICENSE
```

One copy of each skill serves all routes: the repository root is the plugin root, `skills/` is what
the skills CLI scans, and `.mcp.json` registers the server for both plugin systems.

## Verified limits and behaviours

Verified against the live 4Seller MCP endpoint on 2026-11-24. Live schemas and responses remain
authoritative — re-verify after server changes.

- `list_inventory` / `list_inventory_flows`: `skuIdList` is **ignored** by the server; filter by
  `skuId` client-side, or narrow with `warehouseId` and quantity ranges.
- `pageSize` defaults to 20 and caps at 100; page until `page * pageSize >= total`.
- Inventory flow queries span at most 92 days per call.
- An empty result means no record under those filters, never zero stock.
- Saving a price without an ISO 4217 `currency` is rejected.
- `create_listing_draft` is not idempotent; a retry creates a second draft.
- Amazon is out of scope for listing drafts.

## Security

- The skills only reach stores authorized in the 4Seller account they are connected to.
- Stock writes and draft creation require explicit user authorization in the conversation.
- No credentials are stored in this repository; the OAuth sign-in stays with the client that holds
  the token.
- Warehouse and shop IDs differ per tenant — never reuse IDs from examples.

## Publishing this repository

- Keep the repository **public**: `npx skills add` and marketplace import need read access.
- Suggested topics: `agent-skills`, `mcp`, `4seller`, `ecommerce`, `erp`.
- Tag releases (`v0.1.0`) so users can pin.
- Workspace admins can import the marketplace: *Workspace settings → Plugins → Add → Import
  marketplace*, repository URL only.

## License

MIT — see [LICENSE](LICENSE).
