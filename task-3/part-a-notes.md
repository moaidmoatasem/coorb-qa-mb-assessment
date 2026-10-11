# Task 3 — Part A: FX Rates collection

**API used:** Open ExchangeRate API — `https://open.er-api.com/v6/latest/{base_currency}`
**Last run:** 11-Oct-2026 — **51 tests, 51 passed**, 3 iterations (USD, EUR, GBP). Full log in `run-results.md`.

## Files

| File | What it is |
| --- | --- |
| `FX Rates - Banking API.postman_collection.json` | 5 requests and their tests |
| `FX Rates - Banking Env.postman_environment.json` | Base URL, SLA limit, default currency, the three bad codes |
| `rates_data.json` | Base currencies for the data-driven run (USD, EUR, GBP) |
| `run-results.md` | Result of the last full run |

## How to run

Collection Runner → environment `FX Rates - Banking Env` → data file `rates_data.json` → Run.

Or from the command line:

```bash
newman run "FX Rates - Banking API.postman_collection.json" -e "FX Rates - Banking Env.postman_environment.json" -d rates_data.json
```

USD must stay the first row in the data file, because request 2 uses the rate saved on the USD run.

## What the assertions check and why

Think of the bank app showing a customer a rate before they send money abroad. The customer trusts that number. So I check:

| Check | Why it matters |
| --- | --- |
| Status 200 and `result` = `success` | This API can return HTTP 200 with an *error* inside the body (see request 3). Checking only the status is not enough. |
| Response time under 1500 ms | The rate is shown while the customer waits on the transfer screen. 1500 ms is my own assumption; a real SLA would come from the provider contract. |
| `base_code`, `time_last_update_unix`, `rates` exist with the right types | If the shape changes, the app breaks or shows nothing. |
| `base_code` equals the currency we asked for | We must never show EUR rates when the customer asked for USD. |
| Base currency against itself = 1 | A simple sanity check that the numbers are not corrupted. |
| EUR, GBP, EGP, JPY present and above 0 | These are the corridors we care about. A missing or zero rate means a wrong price or a blocked transfer. |
| Data updated in the last 48 hours | The free API refreshes once a day. Older data means the customer sees an old price. |

## The captured value

On the USD run I save `rates.EUR` (USD→EUR) into `captured_usd_to_eur_rate`. Request 2 then gets EUR rates and checks that EUR→USD ≈ 1 ÷ (USD→EUR), within 2%.

**Why this value:** it checks that the provider agrees with itself. If USD→EUR and EUR→USD don't match, a customer could be quoted one price going out and another coming back. The 2% covers rounding and the time between the two calls. (Last run: USD→EUR = 0.892438.)

## Negative cases — what the API actually does

I didn't assume. I tried several bad codes against the live API and found **three different behaviours**, so I test all three:

| Request | Code sent | What the API does | Would I want this in a banking app? |
| --- | --- | --- | --- |
| 3 | `XYZ` — looks like a code, doesn't exist | **HTTP 200** with `{"result":"error","error-type":"unsupported-code"}`, no rates | **No.** An error with a "200 OK" status. A client that checks only the status would treat it as success. I'd want a 4xx (400/422). The `error-type` field itself is good. |
| 4 | `INVALID_CURR` — wrong format | **HTTP 404** with an nginx **HTML** page, not JSON | **Partly.** The status is a clear error, but the format is different from request 3. Our app must handle two error formats from one provider. I'd want one JSON error format for all errors. |
| 5 | `USDX` — too long | **HTTP 200 success with USD rates.** The API quietly uses the first 3 letters (`GBPXYZ` gives GBP rates). | **No — this is the riskiest one.** A typo or bad code gives a real-looking rate for another currency, with no error. Our own app must validate the code (exactly 3 letters, from a known list) **before** calling the provider, and check that `base_code` in the response matches what we asked for — which request 1 already does. |

I assert what the API does today, on purpose. If the provider changes any of these behaviours, the test fails and we notice — because our app's error handling depends on them.

## Limits

- This is a free public API. Rates change daily, so I check ranges and consistency, not exact values.
- Response time also depends on my own network. One slow run is a reason to re-check, not proof of an SLA breach.
