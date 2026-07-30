---
name: leaftfoods-catalog-browse
description: Read Leaft Foods product, collection and store data without authenticating or transacting — via the anonymous Shopify storefront JSON endpoints or the UCP catalog capabilities.
api: leaftfoods:ucp-commerce
generated: '2026-07-19'
method: generated
source: https://www.leaftfoods.com/llms.txt
operations:
  - search_catalog
---

# Browse the Leaft Foods catalog read-only

If you only need to read store data — answer a question about a product, check a price,
summarize the range — do not open the transactional path. The store publishes anonymous
read endpoints for exactly this.

## Anonymous storefront endpoints

No authentication required (documented in the store's `llms.txt`):

| Purpose | Request |
|---|---|
| All products | `GET https://www.leaftfoods.com/collections/all` |
| Product page | `GET /products/{handle}` |
| Product JSON | `GET /products/{handle}.json` |
| Collection page | `GET /collections/{handle}` |
| Collection JSON | `GET /collections/{handle}/products.json` |
| Search | `GET /search?q={query}&type=product` |
| Sitemap | `GET /sitemap.xml` |

Paging uses the Shopify `limit` and `page` query parameters.

> Verified 2026-07-19: `GET /products.json?limit=3` returns HTTP 200 with
> `{"products":[]}` — the endpoint is live and unauthenticated, but the catalog is not
> currently exposed through that path. Prefer `/collections/{handle}/products.json` or the
> UCP `search_catalog` tool, and treat an empty array as "not published here", not as
> "the company sells nothing".

## Via UCP

For structured, agent-shaped catalog access use the MCP endpoint
(`POST https://www.leaftfoods.com/api/ucp/mcp`) and the `search_catalog` tool. The store
declares the `dev.ucp.shopping.catalog.search`, `dev.ucp.shopping.catalog.lookup` and
`dev.shopify.catalog` capabilities at version `2026-04-08`. This requires presenting an
agent profile URI during discovery — see `leaftfoods-agent-purchase.md`.

Pass `context.address_country` and `context.currency` so prices and availability reflect
the buyer's market.

## What the company sells

Leaft Foods extracts Rubisco protein from green leafy crops. Three product lines are
described on the site: the **Leaft Blade** consumer protein beverage, a **Rubisco protein
isolate** ingredient for food manufacturers, and an **Alfalfa Protein Concentrate** for pet
nutrition. Company and product news is published at
`https://www.leaftfoods.com/blogs/news` (Atom: `/blogs/news.atom`).

## Rules

- Respect `robots.txt`, which asks agents to use UCP/MCP for catalog, cart and checkout
  rather than scraping the storefront.
- Back off on 429.
