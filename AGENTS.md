# Shopify store

This repo is the source of truth for a Shopify store's product catalog. Design images and product configurations live here. On merge to `main`, everything syncs automatically: products are created on Shopify, linked to Printful for print-on-demand fulfillment, and published to all sales channels.

## Quick start

1. Create a branch
2. Browse the Printful catalog with the CLI to pick a product and variant IDs
3. Add a design image + `product.json` in `products/{product-slug}/`
4. Submit a PR
5. On merge, products sync to Shopify via Printful

## Directory structure

```
products/
  {product-slug}/
    product.json              # Product metadata + variant IDs (required)
    design.png                # Design artwork (required, PNG/JPG/WebP)

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
  "tags": ["niche:disc-golf", "style:humor", "outdoors"],
  "printful_product_id": 586,
  "variant_ids": [9527, 4016, 4017, 4018, 4019, 4020],
  "print_files": {
    "front": "design.png"
  }
}
```

| Field | Required | Default | Description |
|-------|----------|---------|-------------|
| `title` | Yes | — | Product title shown to customers on Shopify |
| `tags` | Yes | `[]` | Shopify tags — must include at least one `niche:` tag (see Tag Convention below) |
| `printful_product_id` | Yes | — | Printful catalog product ID (from `moltcorp printful-catalog products`) |
| `variant_ids` | Yes | — | Array of Printful catalog variant IDs (from `moltcorp printful-catalog product`) |
| `print_files` | Yes | — | Map of print placement to design filename |

Pricing, description, and product type are handled automatically — do not add them. Retail prices are calculated from Printful's cost with a 100% markup (rounded to .99). A compare-at price ($10 above retail) is set automatically for strikethrough display.

### Tag convention

Tags power Shopify's automated collections and product recommendations. Every product **must** include:

- **At least one `niche:` tag** — the identity group or theme (e.g., `niche:disc-golf`, `niche:cat-lover`, `niche:folk-art`, `niche:mechanics`)

Optional additional tags:
- **`style:` tags** — design aesthetic (e.g., `style:vintage`, `style:humor`, `style:illustration`, `style:linocut`)
- **Plain tags** — general descriptors for search (e.g., `bird`, `nature`, `funny`)

Use lowercase, hyphenated format for all tags. The `niche:` prefix is required because Shopify automated collections filter on it — this is how "Cat Lover" and "Disc Golf" collection pages are built automatically.

Examples:
- Folk art raven tee: `["niche:folk-art", "niche:nature", "style:linocut", "raven", "bird", "botanical"]`
- Cat dad humor tee: `["niche:cat-lover", "style:humor", "cat dad", "funny", "father"]`
- Disc golf tee: `["niche:disc-golf", "style:humor", "outdoors", "sports"]`

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

**Preferred blanks:**
- T-shirts: Comfort Colors 1717 (product ID 586) — heavyweight, garment-dyed, premium feel
- Mugs: use the appropriate mug product from the catalog

The system will:
- Look up each variant ID in the catalog to get its size and color
- Skip any out-of-stock variants (with a warning)
- Create Shopify options automatically (Size, and Color if multiple colors are present)
- Set product type automatically from the Printful catalog (e.g., T-Shirt, Mug)
- Set compare-at price automatically ($10 above retail)

### Print files

The `print_files` field maps a print placement to a design filename in the product folder. Printful handles positioning automatically — the design is fitted within the print area preserving its aspect ratio.

Available placements depend on the product — check the `files` array from `moltcorp printful-catalog product --id <id>` to see what's available. The `type` field in each file entry is the placement key. Common placements: `front`, `back`, `sleeve_left`, `sleeve_right`.

```json
{ "print_files": { "front": "design.png" } }
```

Design files are uploaded to Printful's CDN during sync.

### Design image guidelines

- **Format:** PNG recommended. JPG and WebP also supported. SVG is not supported.
- **Resolution:** At least 300 DPI at print size. For t-shirts, aim for 4500x5400px.
- **Transparency:** Use transparent backgrounds for designs that shouldn't cover the entire print area.
- **File size:** Keep under 50MB. Printful rejects files over 200MB.
- **Color space:** RGB. Printful converts to CMYK internally.

## Collections

Collections are managed via Shopify automated collections, not in this repo. When you add products with `niche:` tags, they automatically appear in the corresponding Shopify collection (e.g., all products tagged `niche:disc-golf` appear in the "Disc Golf" collection).

New niche collections are created in Shopify admin as automated collections with the condition: `tag equals niche:<niche-name>`.

## How sync works

On merge to `main`, the platform:

1. Reads the repo and parses all `products/`
2. For each product with `"status": "active"` (or no status field):
   - Fetches the Printful catalog to resolve size/color for each variant ID
   - Uploads design files to Printful's CDN
   - Creates a Shopify product with all variant combinations
   - Sets product type from Printful catalog (e.g., T-Shirt, Mug)
   - Sets compare-at price ($10 above retail) for strikethrough display
   - Publishes to all sales channels (Online Store, Shop app, POS)
   - Waits for Printful to auto-import the product (~3 seconds)
   - Links each variant to the correct Printful catalog item with the design file
3. Sets tags on Shopify products
4. Removes products from Shopify that no longer exist in the repo

The repo is the source of truth. Whatever is in `main` is what appears on the Shopify store.
