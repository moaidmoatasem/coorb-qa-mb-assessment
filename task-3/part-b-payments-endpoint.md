# Task 3 — Part B: Testing `POST /api/v1/payments`

## 3.1 Highest-risk scenarios

A normal CRUD endpoint can be wrong and you fix the record. A payments endpoint moves money that may not come back. So my focus is: the right person, from the right account, the right amount, exactly once, within the rules.

**Who is paying, and from what**
- The `Authorization` token must own `sourceAccountId`. Test with a valid token and *another customer's* account → must be rejected (403/404), never paid.
- Source account state: active, dormant, frozen, closed, under legal hold → only active can pay.
- Beneficiary state: active, closed, frozen, newly sanctioned → re-checked at payment time, not only when the beneficiary was added.
- Customer segment (retail, premium, corporate) if limits or fees differ by segment.
- `X-Channel` (app, web, branch): same rules and the **same shared limit counter** on every channel.

**The `amount` object**
- **Value:** reject `0`, negative (`-1500.75`), `null`, missing, and a string `"1500.75"` → `400`. A negative amount must never turn into a credit (this happened in ParaBank, Task 2).
- **Precision:** SAR has 2 decimals. `1500.755` must be rejected with `400` — not silently rounded. If the bank decides to round, the rule must be written down and the fee and limit checks must use the same rounded value. Also test currencies with 0 or 3 decimals (JPY, KWD/BHD) if they are supported.
- **Currency:** invalid code (`XYZ`), lower case (`sar`), and a currency different from the source account. Different currency without `fxQuoteId` → rejected. With `fxQuoteId`: expired quote, quote for another customer, quote for another amount → all rejected. A valid quote → the converted amount uses exactly the quoted rate.
- Very large values (`999999999999.99`) and many digits → no overflow, clean `400` or `422`.

**Limits — `422`**
- Per-transaction boundary: `limit - 0.01` ✓, `limit` ✓, `limit + 0.01` → `422`.
- Daily boundary: payments that add up to exactly the daily limit pass, the next `0.01` → `422`.
- Confirm whether the fee counts towards the limit, and test on that rule.
- Failed and reversed payments must not use up the limit.

**Insufficient funds — `402`**
- Check against **available** balance (after holds and pending payments), not the ledger balance.
- Balance covers the amount but not amount + fee → `402`.
- On `402` and `422`: no debit, no fee, no partial record left behind.

**Many requests at the same time**
- Two or more payments sent in parallel (different idempotency keys) that together exceed the balance or the daily limit → only the ones that fit succeed. This is where real money is lost, so I would run this many times, not once.
- Request rate limits per customer → `429`, no payment created.

## 3.2 Idempotency-Key

**What it is for**
A unique key the client creates for **one** payment attempt (e.g. a UUID). If the client sends the same request again — because of a timeout, a retry, or a double tap — the server sees the same key and returns the **original result** instead of paying twice. It also gives us one ID to trace a payment across logs and retries.

**How I would test it**

| Test | Expected |
| --- | --- |
| Same key, same body, sent twice one after the other | Second call returns the same `201` with the same `paymentId` and `reference`. One debit only. |
| Same key, sent twice **in parallel** | One payment created. The other returns the same result, or `409` "in progress". Never two debits. |
| Same key, **different body** (other amount or beneficiary) | `409` (or `422`). The original payment is not changed, no new payment. |
| Same key used by **a different customer** | Treated as a separate key (each customer has their own keys). Customer B must never get customer A's payment response. |
| Different keys, same body, same customer | Two separate payments — this is a real "pay twice" by the customer, and limits and balance still apply. |
| Missing or badly formatted key | `400` — no payment without a key. |
| Key reused after the agreed expiry window (e.g. 24 hours) | Behaviour matches the documented rule. |
| Same key after a failure (`402`, `422`) | Returns the same stored error. To try again after topping up, the client must send a **new** key. |

**Retry after a timeout or a `503`**
- A timeout means the client **doesn't know** if the payment happened. So the client must retry with the **same** key — never a new one.
  - If the first call was processed: the server returns the original `201`. No second debit.
  - If it was never processed: the server processes it now, once.
  - If it is still running: `409` "in progress" — the client waits and retries.
- `503` should mean "not processed". The client retries with the same key, waiting a bit longer between each try. To test it: make the service return `503` (or kill the call at different points — before and after the debit), retry with the same key, and check that the account was debited **exactly once** in every case.

## 3.3 Verifying the money actually moved

A `201` only says the API accepted the request. It may still be `PENDING`. To prove the money moved, I check what was written, not what was returned:

1. **Source account** — balance before and after: decreased by exactly `amount + fee`. Available balance updated, and the hold released if one was used.
2. **Beneficiary** — balance increased by exactly `amount` (internal account), or the outgoing transfer is recorded in the clearing/settlement queue with the right amount and reference (other bank).
3. **Fee** — fee posted to the bank's fee income account, and it matches the `fee` in the response.
4. **FX** — if `fxQuoteId` was used: the converted amount equals amount × the quoted rate, with the agreed rounding.
5. **Status** — the payment moves from `PENDING` to `COMPLETED` within the expected time, and never stays stuck. `valueDate` is correct (holidays, cut-off times).
6. **Ledger (double-entry)** — every payment creates balanced journal entries: debit sender, credit beneficiary or clearing account, credit fee income. Total debits = total credits. This is the same check finance does at end-of-day reconciliation. I worked partly with journal entries on Mambu at Nomo Fintech (a UK digital bank), and this is where "missing" or "created" money shows up.
7. **Everything else that should happen** — exactly one transaction in each account's history with the same `reference`; AML/fraud screening logged as passed; one notification sent to the customer; limit counter increased by the right amount.
8. **Repeat** — after retries (3.2), all checks above still show **one** payment.
