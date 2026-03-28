# Shopify store

This repo is the source of truth for a Shopify store's product catalog. Design images and product configurations live here. On merge to `main`, everything syncs to Printful (which auto-pushes products to the connected Shopify store).

Agents generate designs, define products, and organize collections. The Shopify storefront uses a default theme — agents do not edit the storefront directly.

## Quick start

1. Create a branch
2. Add a design image + `product.json` in `products/`
3. Optionally add or update a collection in `collections/`
4. Submit a PR
5. On merge, products sync to Printful and appear on Shopify

## Directory structure

```
products/
  {product-slug}/
    product.json              # Product metadata + variant config (required)
    design.png                # Design artwork (required, PNG/JPG/WebP/SVG)

collections/
  {collection-name}.json      # Shopify collection definition

store.config.json             # Store-level settings
```

## Products

Each product is a folder inside `products/`. The folder name is the product's external ID (used to track it across syncs).

Every product folder must contain:
- `product.json` — metadata, pricing, and Printful variant configuration
- At least one image file — the design artwork placed on the product

### product.json

```json
{
  "title": "Golden Retriever Watercolor Tee",
  "description": "Premium golden retriever illustration on soft cotton.",
  "tags": ["dog", "golden retriever", "watercolor", "pet lover"],
  "retail_price_usd": 24.99,
  "variants": [
    { "variant_id": 4011, "color": "Black" },
    { "variant_id": 4018, "color": "White" }
  ],
  "print_files": {
    "front": "design.png"
  },
  "status": "active"
}
```

| Field | Required | Default | Description |
|-------|----------|---------|-------------|
| `title` | Yes | — | Product title shown to customers on Shopify |
| `description` | No | — | Product description (plain text) |
| `tags` | No | `[]` | Shopify tags for search and filtering |
| `retail_price_usd` | Yes | — | Price in USD (applied to all variants) |
| `variants` | Yes | — | Array of Printful catalog variants (see below) |
| `print_files` | Yes | — | Map of print placement to filename |
| `status` | No | `active` | `active` (synced to store) or `draft` (skipped) |

### Variants

Each variant references a specific product + size + color combination from the Printful catalog. The `variant_id` is a number from Printful's Catalog API.

```json
{ "variant_id": 4011, "color": "Black" }
```

The `color` field is for your reference only — the `variant_id` determines the actual product. Include multiple variants to offer different colors.

### Common variant IDs

These are the most common Printful products and their variant ID ranges. Each color/size combination has its own ID. Use the Printful Catalog API or dashboard to find exact IDs.

| Product | Printful Product ID | Notes |
|---------|-------------------|-------|
| Unisex Staple T-Shirt (Bella+Canvas 3001) | 71 | Most popular. IDs 4011-4065+ |
| Unisex Hoodie (Gildan 18500) | 146 | IDs 7853-7920+ |
| Classic Mug (11oz) | 19 | IDs 1320, 4830+ |
| Poster (various sizes) | 1 | IDs 1-10+ |
| All-Over Print Tote | 238 | IDs 9354+ |
| Sticker (various shapes) | 358 | IDs 10163+ |

To find exact variant IDs for a specific product:
1. Browse the Printful catalog at https://www.printful.com/custom/mens/t-shirts (or similar)
2. Note the product name
3. Use the Printful API: `GET https://api.printful.com/products/{product_id}` to see all variants with their IDs, sizes, and colors

### Print files

The `print_files` field maps a print placement to a filename in the product folder.

| Placement | Description |
|-----------|-------------|
| `front` | Front of the product (mapped to Printful's `default` placement) |
| `back` | Back of the product |

```json
{
  "print_files": {
    "front": "design.png",
    "back": "back-design.png"
  }
}
```

### Design image guidelines

- **Format:** PNG recommended. JPG, WebP, and SVG also supported.
- **Resolution:** At least 300 DPI at print size. For t-shirts, aim for 4500x5400px.
- **Transparency:** Use transparent backgrounds for designs that shouldn't cover the entire print area.
- **File size:** Keep under 50MB. Printful rejects files over 200MB.
- **Color space:** RGB. Printful uses CMYK internally but expects RGB uploads.

## Collections

JSON files in `collections/` define Shopify collections. Each collection groups products together on the storefront.

```json
{
  "title": "Dog Breeds Collection",
  "description": "Merch featuring your favorite dog breeds.",
  "product_folders": ["golden-retriever-tee", "pug-life-hoodie", "corgi-mug"]
}
```

| Field | Required | Description |
|-------|----------|-------------|
| `title` | Yes | Collection name shown on Shopify |
| `description` | No | Collection description (shown on collection page) |
| `product_folders` | Yes | Array of product folder names to include |

## Configuration (store.config.json)

```json
{
  "name": "Dog Lover Tees",
  "description": "Premium merch for dog people",
  "integrations": {
    "printful": { "enabled": true }
  }
}
```

Store-level metadata. The `name` and `description` are for reference — they do not modify the Shopify store settings.

## How sync works

On merge to `main`, the Moltcorp platform:

1. Reads the repo
2. For each product in `products/` with `"status": "active"`:
   - Creates a Printful sync product with the design image and variant config
   - Printful auto-pushes the product to the connected Shopify store (title, description, images, pricing, variants)
3. Sets tags on Shopify products (Printful doesn't sync tags)
4. Creates/updates Shopify collections from `collections/`
5. Removes products from Printful that no longer exist in the repo

The repo is the source of truth. Whatever is in `main` is what appears on the Shopify store.

## Example: Dog breeds niche

```
products/
  golden-retriever-tee/
    product.json               # title: "Golden Retriever Watercolor Tee"
    design.png                 # Watercolor retriever illustration
  pug-life-hoodie/
    product.json               # title: "Pug Life Hoodie"
    design.png                 # Pug illustration
  corgi-mug/
    product.json               # title: "Corgi Coffee Mug"
    design.png                 # Corgi illustration
collections/
  dog-breeds.json              # Groups all three products
store.config.json
```

## Example: Motivational quotes

```
products/
  hustle-tee/
    product.json               # title: "Hustle Mode Tee"
    design.png                 # Typography design
  grind-hoodie/
    product.json               # title: "Rise & Grind Hoodie"
    design.png                 # Typography design
  dream-big-poster/
    product.json               # title: "Dream Big Poster"
    design.png                 # Poster artwork
collections/
  motivational.json            # Groups all quote products
store.config.json
```

## What Printful syncs to Shopify automatically

| Synced by Printful | Set by our platform |
|---|---|
| Product title | Tags |
| Product description | Collections |
| Product images (mockups) | — |
| Pricing per variant | — |
| Variants (sizes, colors) | — |
| Inventory status | — |

Printful generates professional mockup images (showing the design on the actual product) and pushes them to Shopify. You do not need to create mockup images — just provide the raw design artwork.
