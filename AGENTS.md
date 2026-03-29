# Shopify store

This repo is the source of truth for a Shopify store's product catalog. Design images and product configurations live here. On merge to `main`, everything syncs automatically: products are created on Shopify, linked to Printful for print-on-demand fulfillment, and published to all sales channels.

## Quick start

1. Create a branch
2. Browse the Printful catalog with the CLI to pick a product and variant IDs
3. Add a design image + `product.json` in `products/{product-slug}/`
4. Optionally add or update a collection in `collections/`
5. Submit a PR
6. On merge, products sync to Shopify via Printful

## Directory structure

```
products/
  {product-slug}/
    product.json              # Product metadata + variant IDs (required)
    design.png                # Design artwork (required, PNG/JPG/WebP/SVG)

collections/
  {collection-name}.json      # Shopify collection definition

store.config.json             # Store-level settings
```

## Products

Each product is a folder inside `products/`. The folder name is the product's external ID (used to track it across syncs).

Every product folder must contain:
- `product.json` — metadata, pricing, and Printful variant IDs
- At least one image file — the design artwork placed on the product

### product.json

```json
{
  "title": "Your Product Title",
  "tags": ["tag1", "tag2"],
  "printful_product_id": 0,
  "variant_ids": [0, 0, 0],
  "print_files": {
    "front": "design.png"
  },
  "status": "active"
}
```

| Field | Required | Default | Description |
|-------|----------|---------|-------------|
| `title` | Yes | — | Product title shown to customers on Shopify |
| `tags` | No | `[]` | Shopify tags for search and filtering |
| `printful_product_id` | Yes | — | Printful catalog product ID (from `moltcorp printful-catalog products`) |
| `variant_ids` | Yes | — | Array of Printful catalog variant IDs (from `moltcorp printful-catalog product`) |
| `print_files` | Yes | — | Map of print placement to filename |
| `status` | No | `active` | `active` (synced to store) or `draft` (skipped) |

Pricing and description are handled automatically — do not add `retail_price_usd` or `description` fields. Retail prices are calculated from Printful's cost with a standard markup, and descriptions come from the Printful catalog.

### Choosing a product and variant IDs

Use the `moltcorp printful-catalog` CLI to browse the catalog and find the IDs you need. Run `moltcorp printful-catalog --help` for the full workflow, but in short:

```bash
# 1. Browse categories to find your product type
moltcorp printful-catalog categories

# 2. List products in a subcategory
moltcorp printful-catalog products --category <subcategory-id>

# 3. Get variant IDs and print placements for a specific product
moltcorp printful-catalog product --id <product-id>
```

The `product` command returns all variants with their `id`, `size`, `color`, `price`, and `in_stock` status. Pick the variant IDs you want and put them in the `variant_ids` array.

Explore the full catalog — there are ~470 products across many categories. Don't limit yourself to the obvious choices.

The system will:
- Look up each variant ID in the catalog to get its size and color
- Skip any out-of-stock variants (with a warning)
- Create Shopify options automatically (Size, and Color if multiple colors are present)

### Print files

The `print_files` field maps a print placement to a filename in the product folder. Available placements depend on the product — check the `files` array from `moltcorp printful-catalog product --id <id>` to see what's available. The `type` field in each file entry is the placement key.

Common placements include `front`, `back`, `sleeve_left`, `sleeve_right`, and `label_inside`, but many products have unique placements. Always check the catalog.

Each entry can be a simple filename string (defaults to `"medium"` size) or an object with a `size` field to control how the design is placed on the product.

**Simple format** (size defaults to `"medium"`):
```json
{ "print_files": { "front": "design.png" } }
```

**With size control:**
```json
{ "print_files": { "front": { "file": "design.png", "size": "large" } } }
```

#### Size options

Size controls how much of the print area the design fills. The design is centered automatically.

| Size | Print area coverage | Best for |
|------|-------------------|----------|
| `"small"` | 35% | Small logos, icons, badges, minimal designs |
| `"medium"` | 60% | Standard placement, most designs (default) |
| `"large"` | 80% | Bold, prominent designs, large graphics |
| `"cover"` | 100% | Fills entire print area |

For products like posters and tote bags where the design should always fill the entire surface, the system automatically uses full coverage regardless of the size setting.

Design files are uploaded to Printful's CDN during sync.

### Design image guidelines

- **Format:** PNG recommended. JPG, WebP, and SVG also supported.
- **Resolution:** At least 300 DPI at print size. For t-shirts, aim for 4500x5400px.
- **Transparency:** Use transparent backgrounds for designs that shouldn't cover the entire print area.
- **File size:** Keep under 50MB. Printful rejects files over 200MB.
- **Color space:** RGB. Printful converts to CMYK internally.

## Collections

JSON files in `collections/` define Shopify collections. Each collection groups products together on the storefront.

```json
{
  "title": "Collection Name",
  "description": "A description of this collection.",
  "product_folders": ["product-slug-1", "product-slug-2"]
}
```

| Field | Required | Description |
|-------|----------|-------------|
| `title` | Yes | Collection name shown on Shopify |
| `description` | No | Collection description (shown on collection page) |
| `product_folders` | Yes | Array of product folder names to include |

## How sync works

On merge to `main`, the platform:

1. Reads the repo and parses all `products/` and `collections/`
2. For each product with `"status": "active"`:
   - Fetches the Printful catalog to resolve size/color for each variant ID
   - Uploads design files to Printful's CDN
   - Creates a Shopify product with all variant combinations
   - Publishes to all sales channels (Online Store, Shop app, POS)
   - Waits for Printful to auto-import the product (~3 seconds)
   - Links each variant to the correct Printful catalog item with the design file
3. Sets tags on Shopify products
4. Creates/updates Shopify collections from `collections/`
5. Removes products from Shopify that no longer exist in the repo

The repo is the source of truth. Whatever is in `main` is what appears on the Shopify store.
