# QA Technical Assessment — Moayed Badawy

Level: **Lead** (all five tasks completed).

| Folder | Task | Main file |
| --- | --- | --- |
| `task-1/` | Test Design & Risk Coverage — Local Fund Transfer | `task-1.md` |
| `task-2/` | Exploratory Testing & Bug Reporting — ParaBank | `task-2.md` + `evidence/` |
| `task-3/` | API Testing — Part A (Postman) and Part B (payments endpoint) | `part-a-notes.md`, collection, environment, data file, `part-b-payments-endpoint.md` |
| `task-4/` | Automation Design — Bank Operations Console | `task-4.md` |
| `task-5/` | Quality Strategy & Release Judgement | `task-5-release-memo.md` |
| — | How I used AI tools | `ai-usage.md` |

## Time spent

| Task | Time |
| --- | --- |
| Task 1 | 1 h 15 min |
| Task 2 | 50 min |
| Task 3 | 1 h 10 min |
| Task 4 | 45 min |
| Task 5 | 30 min |
| Packaging (repo, evidence, README) | 40 min |
| **Total** | **about 5 h 10 min** |

This is more than the suggested 2.5–3 hours. Most of the extra time went into Task 1 (breaking down the requirement AC by AC before writing cases) and into checking the FX API's real behaviour with bad currency codes instead of assuming it.

## Environments and tools used

- **Task 2:** ParaBank primary site (https://parabank.parasoft.com) — reachable on 09-Oct-2026; backup not needed.
- **Task 3 Part A:** Open ExchangeRate API (`https://open.er-api.com/v6/latest/{base}`). Last full run: 51/51 tests passed (`task-3/run-results.md`). Can also run with Newman — command in `task-3/part-a-notes.md`.

## Main assumptions

The full list of questions and assumptions for the transfer feature is in `task-1/task-1.md` (section 1.1). The ones that shape my test design most:

- Source and beneficiary use the **same local currency**; no FX in the Local Fund Transfer feature.
- The **fee is added on top** of the amount and does **not** count towards the 10,000 / 20,000 limits; the balance must cover amount + fee.
- The **daily limit is per customer, shared across channels**, and resets at midnight bank time. Failed and reversed transfers don't use it up.
- **OTP by SMS**, 6 digits, 5-minute expiry, a new OTP cancels the old one, 3 wrong attempts cancel the transfer and lock transfers (not the whole profile) for a period.
- Beneficiary status and **sanctions are re-checked at execution**, not only at registration.
- After a timeout, a transfer must end **completed or fully reversed** — never half-done.

## Questions I would ask the BA / PO first

1. What currency are the limits in, and are cross-currency transfers in scope?
2. Is the fee on top of the amount, and does it count towards the limits?
3. When does the "day" reset, and is the limit per customer or per account, shared across channels?
4. OTP: delivery channel, expiry, retry limit, and exactly what gets locked?
5. If the connection drops right after the OTP, does the money come back automatically, or does the transfer show as pending — and who in operations fixes it?
6. Is the beneficiary re-checked against sanctions and account status at execution time?

## What I deliberately left out, and why

- **Task 1:** FX / cross-currency, self-transfers, beneficiary registration, and performance / DR testing. They're either out of scope by my assumptions or need their own plan. Reasons are in section 1.4.
- **Task 2:** I found more than two issues, but I reported the two with the clearest expected behaviour and different root causes. The others are listed briefly in section 2.2. I didn't explore the open Administration page further, because security testing was out of scope.
- **Task 3 Part A:** I didn't assert exact rate values — the free API updates once a day, so I check ranges, freshness and consistency instead.
- **Task 4:** `Bank_Admin_Console_Demo.html` was not in the pack I received, so I designed from the written description of the console. No code was implemented, as the brief asked.
- **Task 5:** I kept the memo within one and a half pages, so some detail (e.g. the exact FX test list) is summarised.
