# kari catalog MCP server

Public [Model Context Protocol](https://modelcontextprotocol.io) server for the catalog of
[kari](https://kari.com/), a chain of shoe, clothing and accessory stores in Russia, Kazakhstan
and Belarus. AI assistants use it to search products, check prices, find stores and see
which store has a size in stock.

```
https://mcp.kari.com/mcp
```

- Transport: Streamable HTTP, stateless, JSON responses.
- No authentication, no API key.
- Read-only: the server never places orders and has no cart. Purchase and pickup happen on
  the website through the product `url`.
- Rate limit: 60 requests per minute per IP address.
- Registry name: `com.kari/catalog` in the
  [official MCP Registry](https://registry.modelcontextprotocol.io/v0.1/servers?search=com.kari).

This repository holds the description, client configuration and an agent skill. The server
itself runs at kari and is not open source.

## What you can ask

- "Find white sneakers in size 38 under 5000 rubles in Kazan"
- "Is there a store in Almaty where I can pick up this model in size 41 today?"
- "Show kari stores in Minsk with their opening hours"
- "Compare the composition and rating of these two boots"
- "Найди женские ботинки 37 размера в Москве со скидкой"

## Connect

### Claude (claude.ai, desktop and mobile apps)

Settings → Connectors → **Add custom connector**, URL `https://mcp.kari.com/mcp`.
No OAuth settings are needed.

### Claude Code

```bash
claude mcp add --transport http kari https://mcp.kari.com/mcp
```

Or install the plugin from this repository, which adds the server and the skill together:

```
/plugin marketplace add kari-it/kari-mcp
/plugin install kari-catalog@kari
```

### ChatGPT

Settings → Apps & Connectors → Advanced → enable **Developer mode**, then create a connector
with the URL `https://mcp.kari.com/mcp` and authentication **No authentication**.

### VS Code / GitHub Copilot

`.vscode/mcp.json`:

```json
{
  "servers": {
    "kari": { "type": "http", "url": "https://mcp.kari.com/mcp" }
  }
}
```

### Cursor

`~/.cursor/mcp.json`:

```json
{
  "mcpServers": {
    "kari": { "url": "https://mcp.kari.com/mcp" }
  }
}
```

### Other clients

Any client that supports remote MCP servers over Streamable HTTP. For clients that only
speak stdio, use a bridge such as [`mcp-remote`](https://www.npmjs.com/package/mcp-remote):

```json
{
  "mcpServers": {
    "kari": { "command": "npx", "args": ["-y", "mcp-remote", "https://mcp.kari.com/mcp"] }
  }
}
```

## Tools

Every tool takes `country`: `ru` (Russia, RUB), `kz` (Kazakhstan, KZT) or `by` (Belarus, BYN).
Prices and stock depend on the city; without `city_id` the capital is used.

| Tool | What it does | Key inputs |
|---|---|---|
| `find_city` | Finds a city where kari operates by part of its name; returns up to 10 cities with `city_id` | `query` |
| `find_stores` | Stores in a city with address, opening hours and coordinates; filter by name, address, mall or metro; 10 per page | `city_id`, `query`, `page` |
| `search_products` | Catalog search with filters; returns price, size range and a link to the website | `query`, `city_id`, `price_min`, `price_max`, `size`, `brand`, `sort`, `limit`, `page` |
| `get_product` | Product details: price, images, color, composition, rating, sizes with availability in the city | `articul`, `city_id` |
| `check_availability` | Stores in the city that can hand over the size, and when (pickup); quantities are not disclosed | `articul`, `size`, `city_id` |

`sort`: `relevance`, `price_asc`, `price_desc`, `rating`, `discount`.

All tools are annotated `readOnlyHint: true`, `destructiveHint: false`.

### Typical flow

1. `find_city` with the user's city, keep `city_id`.
2. `search_products` with the request in the user's language and filters.
3. `get_product` for the chosen `articul`: sizes, composition, `available`.
4. `check_availability` for the size: which stores can hand it over and when.
5. Give the user the product `url`; ordering happens on the website.

Notes for models:

- `sizes` in `search_products` is the size range of the model, not stock.
- `available` in `get_product` means the item can be ordered in this city (store or
  warehouse), not that it is on a shelf today. Pickup timing comes from `check_availability`.
- Items without sizes (bags, accessories) have an empty `sizes`; call `check_availability`
  without `size`.
- `old_price` and `discount_percent` are present only when the website shows a markdown.

## Calling without an MCP client

Send one JSON-RPC 2.0 message per POST request; batches are not supported and no session
header is needed.

```bash
curl -X POST https://mcp.kari.com/mcp \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"search_products","arguments":{"country":"ru","query":"белые кроссовки","limit":3}}}'
```

A plain-text description for models is served at <https://mcp.kari.com/>.

## Agent skill

[`skills/kari-catalog/SKILL.md`](skills/kari-catalog/SKILL.md) is an
[Agent Skill](https://agentskills.io) that tells an agent when to use this server and in what
order to call the tools. Claude Code picks it up through the plugin above; other agents can
copy the folder into their skills directory.

## Discovery

- `https://kari.com/llms.txt`, `https://kari.com/ai/` — description for people and models.
- `https://mcp.kari.com/.well-known/mcp/server-card.json` and
  `https://kari.com/.well-known/mcp.json` — server card.
- [`server.json`](server.json) — the entry published in the MCP Registry.

## На русском

Открытый MCP-сервер каталога kari: поиск обуви, одежды и аксессуаров, цены, магазины и
наличие размера в России, Казахстане и Беларуси. Адрес `https://mcp.kari.com/mcp`,
авторизация не нужна, только чтение, лимит 60 запросов в минуту с одного адреса.
Инструкции по подключению — на странице <https://kari.com/ai/>.

## Contact

Questions about the server: <https://kari.com/ai/>.
