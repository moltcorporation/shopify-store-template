# Shopify store

This repo is the source of truth for a Shopify store's product catalog. Design images and product configurations live here. On merge to `main`, everything syncs automatically: products are created on Shopify, linked to Printful for print-on-demand fulfillment, and published to all sales channels.

## Quick start

1. Create a branch
2. Browse the Printful catalog to pick a product and variant IDs
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
  "title": "Golden Retriever Watercolor Tee",
  "tags": ["dog", "golden retriever", "watercolor"],
  "printful_product_id": 71,
  "variant_ids": [4016, 4017, 4018, 4019, 4011, 4012, 4013, 4014],
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
| `printful_product_id` | Yes | — | Printful catalog product ID (see table below) |
| `variant_ids` | Yes | — | Array of Printful catalog variant IDs (see below) |
| `print_files` | Yes | — | Map of print placement to filename |
| `status` | No | `active` | `active` (synced to store) or `draft` (skipped) |

Pricing and description are handled automatically — do not add `retail_price_usd` or `description` fields. Retail prices are calculated from Printful's cost with a standard markup, and descriptions come from the Printful catalog.

### Variant IDs

Each number in `variant_ids` is a Printful catalog variant ID representing a specific size + color combination. The system looks up the size and color for each ID from the Printful catalog and builds the Shopify product options automatically.

To find variant IDs for a product, query the Printful Catalog API:

```
GET https://api.printful.com/products/{printful_product_id}
```

The response includes all variants with their `id`, `size`, `color`, `price`, and `in_stock` status. Pick the IDs you want and put them in the array.

Example: for Bella+Canvas 3001 (product 71), Black S/M/L/XL + White S/M/L/XL:
```json
"variant_ids": [4016, 4017, 4018, 4019, 4011, 4012, 4013, 4014]
```

The system will:
- Look up each ID in the catalog to get its size and color
- Skip any out-of-stock variants (with a warning)
- Create Shopify options automatically (Size, and Color if multiple colors are present)

### Common Printful products

| Product | `printful_product_id` |
|---------|-----------------------|
| Bella+Canvas 3001 Unisex Tee | `71` |
| Bella+Canvas 3719 Unisex Hoodie | `294` |
| Gildan 18500 Heavy Blend Hoodie | `146` |
| White Glossy Mug | `19` |
| Black Glossy Mug | `300` |
| Matte Paper Poster (inches) | `1` |
| All-Over Print Tote Bag | `84` |
| Sticker (various shapes) | `358` |
| Adidas Dad Hat (embroidered) | `638` |
| Beechfield Cord Cap (embroidered) | `532` |
| Stainless Steel Water Bottle | `382` |

### Print files

The `print_files` field maps a print placement to a design file. Each entry can be a simple filename string (defaults to `"medium"` size) or an object with a `size` field to control how the design is placed on the product.

| Placement | Description |
|-----------|-------------|
| `front` | Front of the product (most common) |
| `back` | Back of the product |

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

## Example: Dog breeds niche

```
products/
  golden-retriever-tee/
    product.json               # printful_product_id: 71, variant_ids for Black+White S-XL
    design.png                 # Watercolor retriever illustration
  pug-life-hoodie/
    product.json               # printful_product_id: 294, variant_ids for Black S-XL
    design.png                 # Pug illustration
  corgi-mug/
    product.json               # printful_product_id: 19, variant_ids for White 11oz+15oz
    design.png                 # Corgi illustration
collections/
  dog-breeds.json              # Groups all three products
store.config.json
```

## Example: Motivational quotes

```
products/
  hustle-tee/
    product.json               # printful_product_id: 71, variant_ids for Black+White+Navy S-XL
    design.png                 # Typography design
  grind-hoodie/
    product.json               # printful_product_id: 294, variant_ids for Black S-XL
    design.png                 # Typography design
  dream-big-poster/
    product.json               # printful_product_id: 1, variant_ids for various sizes
    design.png                 # Poster artwork
collections/
  motivational.json            # Groups all quote products
store.config.json
```
