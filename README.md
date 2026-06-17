# PicoWAL Retail Primitives

Retail-specific catalog, search, recommendation, event, PicoWeb API, and storefront primitives built on top of the generic PicoWAL stack.

This repository deliberately keeps retail behavior **outside** the baseline primitive repositories:

| Repository | Responsibility |
| --- | --- |
| `picowal` | Generic pack/card storage, host stores, search/index primitives, index persistence, journal replay |
| `picoweb` | Generic route matching and request/response dispatch |
| `BareMetalJsTools` | Generic browser-side modules such as REST transport and local search helpers |
| `picowal.retailprimitives` | Retail-specific product catalog, search/recommend API shape, event capture, and demo storefront |

## What these primitives provide

### Product catalog storage

`picowal_retail` stores products as bounded PicoWAL cards. A product has fixed-size fields suitable for a compact server-side retail catalog:

```c
typedef struct {
    char id[32];
    char title[96];
    char description[160];
    char category[32];
    char brand[32];
    char tags[96];
    float price;
    uint32_t inventory;
} picowal_retail_product_t;
```

Product IDs are deterministically mapped to PicoWAL card IDs with `picowal_retail_card_for_id()`.

### Search indexing

Every product upsert updates the underlying `picowal_search` index:

- product text index: title, description, category, brand, tags
- vector signature index: deterministic local text embedding
- category and brand facet entries
- price numeric/range entry
- append-only search journal stored in PicoWAL
- persistent index segment stored in PicoWAL

This keeps **catalog data, derived indexes, and recovery journal inside PicoWAL**.

### JSON API helpers

The core library can produce bounded JSON responses for simple host/server APIs:

- `picowal_retail_products_json()`
- `picowal_retail_product_json()`
- `picowal_retail_search_json()`
- `picowal_retail_recommend_json()`

### PicoWeb route table

`picowal_retail_web` exposes a `picoweb_route_table_t`:

| Method | Route | Purpose |
| --- | --- | --- |
| `GET` | `/` | demo storefront |
| `GET` | `/retail` | demo storefront |
| `POST` | `/api/retail/products:ingestDemo` | seed bundled demo products |
| `GET` | `/api/retail/products` | list products |
| `GET` | `/api/retail/products/{id}` | product detail |
| `POST` | `/api/retail/search` | search products |
| `POST` | `/api/retail/recommend` | recommend similar products |
| `POST` | `/api/retail/events` | record a user event |

### BareMetalJsTools storefront

`host/www/retail_storefront.inc` is an embedded HTML asset that uses:

- `BareMetal.Communications` for API calls
- `BareMetal.Search` as a browser-side helper module

It is intentionally stored in this retail-primitives repo, not in `BareMetalJsTools`.

## Storage layout

You choose the packs/cards when initializing the retail context:

```c
picowal_retail_init(&retail,
    8,    // product_pack
    9,    // index_pack
    100,  // index_base_card
    10,   // journal_pack
    200   // journal_base_card
);
```

Recommended layout:

| Data | Suggested location |
| --- | --- |
| Products | one user pack, e.g. pack `8` |
| Search index segment | dedicated pack/base card, e.g. pack `9`, base card `100` |
| Search journal | dedicated pack/base card, e.g. pack `10`, base card `200` |
| Events | separate pack, e.g. pack `11` |

The product pack contains product cards. The index and journal use the generic PicoWAL search pack/card persistence primitives from `picowal_search`.

## Core usage

```c
#include "picowal_api.h"
#include "picowal_retail.h"
#include "picowal_store_fs.h"

picowal_fs_store_t fs;
picowal_store_t store;
picowal_store_fs_open(&fs, "/tmp/retail-picowal", &store);
picowal_api_set_store(&store);

picowal_retail_t retail;
picowal_retail_init(&retail, 8, 9, 100, 10, 200);

picowal_retail_product_t product = {
    .id = "aurora-shell",
    .title = "Aurora Storm Shell Jacket",
    .description = "Breathable waterproof shell with taped seams",
    .category = "outerwear",
    .brand = "Contoso Trail",
    .tags = "waterproof hiking rain",
    .price = 129.99f,
    .inventory = 142,
};

picowal_retail_upsert(&retail, &product);
```

## Search usage

```c
char json[PICOWAL_RETAIL_JSON_MAX];

picowal_retail_search_json(&retail,
    "waterproof jacket",
    json,
    sizeof(json)
);
```

Example response shape:

```json
{
  "results": [
    {
      "id": "aurora-shell",
      "score": 1.2345,
      "product": {
        "id": "aurora-shell",
        "title": "Aurora Storm Shell Jacket",
        "category": "outerwear",
        "brand": "Contoso Trail",
        "price": 129.99
      }
    }
  ],
  "totalSize": 1,
  "facets": {
    "category": [{ "value": "outerwear", "count": 2 }]
  }
}
```

## Demo data

Seed the built-in demo catalog:

```c
picowal_retail_ingest_demo(&retail);
```

The demo includes outdoor retail products such as waterproof jackets, boots, base layers, packs, lanterns, and down jackets.

## PicoWeb integration

```c
#include "picowal_retail_web.h"

picowal_retail_web_t web = {
    .retail = &retail,
    .event_pack = 11,
};

picoweb_dispatch(&picowal_retail_route_table, &request, &response, &web);
```

The route table is intentionally generic PicoWeb-compatible C. It does not require a framework-specific HTTP server.

## Build and test

Expected sibling checkout layout:

```text
C:\source\picowal
C:\source\picoweb
C:\source\picowal.retailprimitives
```

Build with CMake:

```bash
cmake -S . -B build
cmake --build build
./build/picowal_retail_smoke
```

Or compile directly with GCC:

```bash
gcc -std=c11 -Wall -Wextra -Werror \
  -DPICOWAL_HOST=1 -DPICOWAL_NO_DEFAULT_STORE=1 \
  -Isrc -I../picowal/src -I../picoweb/include \
  ../picowal/src/picowal_api.c \
  ../picowal/src/picowal_search.c \
  ../picowal/src/picowal_store_fs.c \
  src/picowal_retail.c \
  src/picowal_retail_web.c \
  host/tests/retail_smoke.c \
  -lm -o picowal_retail_smoke

./picowal_retail_smoke
```

Expected output:

```text
picowal retail smoke ok
```

## Boundary rules

- Add reusable storage/index primitives to `picowal`.
- Add generic HTTP dispatch behavior to `picoweb`.
- Add generic browser modules to `BareMetalJsTools`.
- Add retail-specific catalog/search/recommend/event/frontend behavior here.

