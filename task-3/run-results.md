# Run results — FX Rates - Banking API

- **Date:** 11-Oct-2026, 00:57 UTC
- **Environment:** FX Rates - Banking Env (`https://open.er-api.com`, SLA 1500 ms)
- **Data file:** `rates_data.json` — 3 iterations: USD, EUR, GBP
- **Result:** **51 tests — 51 passed, 0 failed**
- **Captured value:** USD→EUR = 0.892438

| Iteration | Request | HTTP | Time | Tests |
| --- | --- | --- | --- | --- |
| 1 (USD) | 1. Get latest rates | 200 | 250 ms | 7/7 ✅ |
| 1 (USD) | 2. EUR→USD inverse check | 200 | 51 ms | 3/3 ✅ |
| 1 (USD) | 3. Unsupported code `XYZ` | 200 | 72 ms | 3/3 ✅ |
| 1 (USD) | 4. Malformed code `INVALID_CURR` | 404 | 51 ms | 2/2 ✅ |
| 1 (USD) | 5. Too-long code `USDX` | 200 | 51 ms | 2/2 ✅ |
| 2 (EUR) | 1. Get latest rates | 200 | 49 ms | 7/7 ✅ |
| 2 (EUR) | 2. EUR→USD inverse check | 200 | 50 ms | 3/3 ✅ |
| 2 (EUR) | 3. Unsupported code `XYZ` | 200 | 55 ms | 3/3 ✅ |
| 2 (EUR) | 4. Malformed code `INVALID_CURR` | 404 | 50 ms | 2/2 ✅ |
| 2 (EUR) | 5. Too-long code `USDX` | 200 | 49 ms | 2/2 ✅ |
| 3 (GBP) | 1. Get latest rates | 200 | 52 ms | 7/7 ✅ |
| 3 (GBP) | 2. EUR→USD inverse check | 200 | 51 ms | 3/3 ✅ |
| 3 (GBP) | 3. Unsupported code `XYZ` | 200 | 51 ms | 3/3 ✅ |
| 3 (GBP) | 4. Malformed code `INVALID_CURR` | 404 | 50 ms | 2/2 ✅ |
| 3 (GBP) | 5. Too-long code `USDX` | 200 | 53 ms | 2/2 ✅ |

Note: requests 2–5 don't use the data file, so they run once per iteration with the same result. In a real suite I'd put them in a separate folder and run them once; I kept one flat collection here to keep it simple to read.
