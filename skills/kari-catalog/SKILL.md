---
name: kari-catalog
description: Search the kari catalog (shoes, clothing, bags, accessories) in Russia, Kazakhstan and Belarus through the kari MCP server — products, prices, sizes, stores and pickup availability. Use when the user asks about kari / кари, wants to find or compare shoes or clothing at kari, check a price, find a kari store, or see where a size can be picked up.
---

# kari catalog

kari is a chain of shoe, clothing and accessory stores in Russia, Kazakhstan and Belarus.
The public MCP server `https://mcp.kari.com/mcp` (Streamable HTTP, no authentication,
read-only) answers catalog questions with live prices and stock.

If the kari tools are not connected, tell the user how to add the server:
Claude Code — `claude mcp add --transport http kari https://mcp.kari.com/mcp`;
other clients — see https://kari.com/ai/.

## Pick the country

`country` is required in every call: `ru` (Russia, RUB), `kz` (Kazakhstan, KZT),
`by` (Belarus, BYN). Take it from the user's city, currency or language; ask if unclear.

## Call order

1. **City.** `find_city` with part of the city name. Keep `city_id` and pass it to every
   later call: prices and stock depend on the city. Without it the capital is used.
2. **Search.** `search_products` with the request in the user's language
   (for example `"женские ботинки"`), plus `size`, `price_min`, `price_max`, `brand`,
   `sort` (`relevance`, `price_asc`, `price_desc`, `rating`, `discount`). Page with `page`;
   `limit` up to 20.
3. **Details.** `get_product` with the `articul`: images, color, composition, rating and
   `sizes` with `available` per size in the city.
4. **Pickup.** `check_availability` with `articul`, `size` and `city_id`: which stores can
   hand the item over and when. Quantities are not disclosed.
5. **Stores.** `find_stores` for addresses, opening hours and coordinates; `query` filters
   by name, address, mall or metro station.

## Read the results correctly

- `sizes` in `search_products` is the model's size range, not stock. Confirm a size with
  `get_product` or `check_availability` before telling the user it is available.
- `available: true` means the item can be ordered in the city (store or warehouse). It does
  not mean it is on a shelf today; pickup timing comes from `check_availability`.
- Items without sizes (bags, accessories) have an empty `sizes`; call
  `check_availability` without `size`.
- `old_price` and `discount_percent` appear only when the website shows a markdown.
  Quote prices in the country currency.
- Always give the product `url`. The server has no cart: ordering, payment and pickup
  booking happen on the website.

## Errors

- "City not found" — call `find_city` again and use its `city_id`.
- "Product not found" — use an `articul` from `search_products`.
- "Size not found" — take a size from `get_product`; omit it for items without sizes.
- "kari catalog is temporarily unavailable" — retry later and tell the user.

The server allows 60 requests per minute per IP address. Avoid calling `get_product` for
every search result; fetch details only for the products the user is interested in.
