# PicoWAL Retail Primitives

Retail-specific primitives built on top of the generic PicoWAL, PicoWAL Search,
PicoWeb, and BareMetalJsTools baseline primitives.

This library intentionally lives outside `picowal` and `BareMetalJsTools`:

- `picowal` owns generic pack/card storage and search/index primitives.
- `picoweb` owns generic route dispatch.
- `BareMetalJsTools` owns generic frontend modules.
- `picowal.retailprimitives` owns retail catalog/search/recommend/event behavior.

## Contents

- `src/picowal_retail.*` — bounded product catalog stored in PicoWAL cards,
  indexed through `picowal_search`.
- `src/picowal_retail_web.*` — PicoWeb-compatible retail API route table.
- `host/www/retail_storefront.inc` — BareMetalJsTools storefront asset.
- `host/tests/retail_smoke.c` — host smoke test.

## Build/test

```bash
cmake -S . -B build
cmake --build build
./build/picowal_retail_smoke
```

The smoke test compiles against sibling checkouts:

- `../picowal`
- `../picoweb`
