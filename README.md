# 4Seller Agent Skills

Agent Skills that let an AI assistant work with a seller's 4Seller ERP account through the
4Seller MCP server — inventory and listing drafts, with explicit read/write boundaries.

Install from this repository by URL. The repository is simultaneously a **Claude Code plugin**, a
**Codex plugin**, and a **skills-CLI package**, so every route ships the same two skills. The plugin
routes also declare the 4Seller MCP server (`.mcp.json`), so installing is what connects it.

## Included skills

| Skill | Capabilities | Writes? | MCP tools used |
|---|---|---|---|
| `4seller-inventory` | Current stock quantity, inventory flow history, stock adjustments | `adjust_inventory` only, with a read-before-write check | `list_stores`, `list_inventory`, `list_inventory_flows`, `adjust_inventory` |
| `4seller-listing` | Convert listing raw data (Excel/CSV/XLSX, product URLs, pasted data, API output) into listing drafts via the crawl box | `save_crawl_draft`, `create_listing_draft` | `list_stores`, `save_crawl_draft`, `create_listing_draft`, `get_draft_status` |

Neither skill publishes to a marketplace; publishing stays a user action inside 4Seller.

## Install

### Claude Code plugin (skills + MCP connection)

```text
/plugin marketplace add zhen123456/4seller-agent-skills
/plugin install 4seller@4seller-agent-skills
```

The plugin declares the server in `.mcp.json`; finish with the sign-in at `/mcp`.
Skills are namespaced: `/4seller:4seller-inventory`.

### Codex plugin (skills + MCP connection, auth at install)

Open the repository in Codex, then run `/plugins` and install **4Seller Agent Skills**. The
marketplace entry sets `authentication: ON_INSTALL`, so the 4Seller OAuth prompt runs as part of the
install. Command-line equivalents:

```bash
codex plugins list          # confirm the repo marketplace is visible
codex plugins install 4seller
```

Codex reads the repo marketplace at `.agents/plugins/marketplace.json`, and also accepts a
Claude-compatible `.claude-plugin/marketplace.json`. Confirm the server afterwards with
`codex --print-tools` or `/mcp` inside a session.

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

## Verify

Start a new session and ask: *"How much stock is available for SKU X?"* or
*"Turn the products in this spreadsheet into drafts for my Shopify shop."*

## One install channel per machine

Identity here is name-based (no opaque IDs): marketplace `4seller-agent-skills`, plugin `4seller`,
skills `4seller-inventory` / `4seller-listing`, MCP server `4seller-prod`.

| Situation | What happens |
|---|---|
| Re-installing the same plugin | Replaces it — one slot per marketplace + plugin name |
| Re-running `npx skills add` | Overwrites `<skills-dir>/4seller-inventory/` |
| Plugin install **plus** a manual copy | Duplicate: one copy under the plugin namespace (`/4seller:4seller-inventory`), one in the skills directory |
| The same repository added as two marketplaces | Two markets, two same-named plugins — remove one |

Pick one route per machine and stay on it. To inspect or clean up:

```text
/plugin                                          # Claude Code: installed plugins
/plugin uninstall 4seller@4seller-agent-skills
codex plugins list                               # Codex
codex plugins remove 4seller
npx skills list                                  # skills CLI
npx skills remove 4seller-inventory 4seller-listing
```

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
