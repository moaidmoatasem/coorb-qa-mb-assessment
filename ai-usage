# How I used AI tools

I used AI as a thinking partner and a writing assistant. I did the first thinking on every task myself, then used AI to find gaps, organise, and tighten the wording. Every decision in the final submission is mine, and I can explain each one.

## Tools

| Tool | Used for |
| --- | --- |
| **Google Gemini** | Coaching while I broke down the requirement (Task 1), organising my notes, first drafts of Tasks 3B, 4 and 5. |
| **An AI coding agent (opencode)** | Running exploratory checks on ParaBank's API in parallel with my own manual exploration, and recording request/response evidence. |
| **Postman AI (Postbot)** | Creating the collection and environment in my Postman workspace from my design. |
| **Claude** | Reviewing the exported Postman files, fixing the bugs it found, simplifying the scripts, running the collection's test scripts against the live API, and final editing of all documents into plain English. |

## Main prompts (summarised)

- **Task 1 (Gemini):** *"I would break down my thoughts with you to keep it fair, with minimal help from your end (you may hint in case I missed something major)."* — then I posted my own questions AC by AC. Later: *"Break down the questions per each AC as I did, rephrase my questions, add questions I missed."* I added my own points along the way (self-transfers and same-account transfers, OTP delivery channel, negative/zero amounts, OTP rate limiting, peak load and outages).
- **Task 2 (opencode agent + me):** the agent explored ParaBank's money flows through the screens and the backend, and recorded the evidence — including the screenshots in `task-2/evidence/`. I reviewed every screenshot and the numbers in it before using them. Separately I explored by hand and found the same pattern: transfer to the same account, negative and zero amounts, more than the balance, no validation on Bill Pay / Request Loan / Update Contact Info, an open Administration page. Gemini then helped me pick and format the two bugs.
- **Task 3 Part A (Gemini → Postbot → Claude):** Gemini drafted the scripts, Postbot built the collection, and I asked Claude to *"make it simpler, straightforward automation scripts with no complexity, as I have been away from coding for a while."*
- **Task 3 Part B (Gemini):** I wrote my own bullets for 3.1–3.3 first (auth and account ownership, account state, limits, race conditions, currencies, idempotency-key race tests with same/different users, retries after failures, verifying debit/credit/fees/FX/AML), then asked for them to be organised.
- **Tasks 4 and 5 (Gemini, then Claude):** first drafts, then rewritten against the exact brief.

## What I kept

- My own question breakdown for Task 1, organised by acceptance criterion with "why it matters" and an assumption for each.
- Risk-based test cases written as scenarios, not one case per field — the 12-case cap needs that.
- My Part B bullets, organised into tables and clear expected results.
- The idea of checking journal entries (double-entry) to prove money moved — I worked partly with this on Mambu at Nomo Fintech (a UK digital bank).

## What I rejected or corrected, and why

- **Wrong claims about the FX API's errors.** The Gemini draft said an invalid currency returns HTTP 200 with an error body. I didn't take it on trust and ran the API live with several bad codes. The truth has three cases: `XYZ` → 200 with an error body; `INVALID_CURR` → 404 with an HTML page; `USDX` → **200 success with USD rates** (the API uses the first 3 letters). I built one negative test for each, and wrote whether each is acceptable for a bank.
- **Bugs in the Postbot collection.** A pre-request script forced the currency to `usd`, which overrode the data file — so EUR and GBP were never really tested, and the base-currency check would fail. The environment export also had empty values. Both fixed.
- **Over-complex scripts.** I removed loops, fallbacks and try/catch, so each test is a few plain lines I can explain.
- **Which two bugs to report.** Gemini recommended the Request Loan "$0 down" issue as a main bug. I rejected that: ParaBank doesn't define its lending rules, so my "expected result" would be my opinion, not the app's spec. I picked two bugs where the expected behaviour comes from the app itself plus clear banking rules. The loan issue is listed under other findings, with a question for the PO.
- **An unverified bug.** A draft described "Find Transactions is broken" with guessed symptoms. I didn't report it as a bug — it's in my "next two hours" list instead.
- **A generic Task 4 draft.** The reference HTML page was not attached to the email, and Gemini filled the gap with a generic admin-console plan (RBAC, audit logs) that did not answer the six required areas. I rewrote it from the written brief around three simple ideas: find fields by label not by dynamic ID, one set/get building block for all control types, and fields kept as data. Plus back-up-and-restore for the shared environment.
- **Wording.** I removed over-confident language ("100% safe", "guaranteed") and kept the tone plain.
