# Task 1 — Test Design & Risk Coverage: Local Fund Transfer

## 1.1 Questions and assumptions

I grouped the questions by acceptance criterion, so it is clear which part of the requirement each gap comes from. For each one: why it matters, and the assumption I test against if I get no answer.

### Before AC1 — scope

| # | Question | Why it matters | My assumption if no answer |
| --- | --- | --- | --- |
| Q1 | Is this flow for retail customers only, or do corporate users share it? | Corporate users usually have different limits, approvals (maker/checker) and fees. | Retail only. |
| Q2 | Does "beneficiary" include the customer's own other accounts (self-transfer), or only third parties? Can source and beneficiary be the same account? | Self-transfers often have no fee, no limit, and different rules. Same-account transfer must be blocked. | Third-party beneficiaries only. Same account is not selectable. |

### AC1 — Transfer initiation

| # | Question | Why it matters | My assumption if no answer |
| --- | --- | --- | --- |
| Q3 | What is the currency of the accounts and the limits? Can source and beneficiary be in different currencies? | "10,000" means nothing without a currency. Cross-currency brings in FX rates, rounding and rate locking. | Same local currency on both sides. No FX in this feature. |
| Q4 | What are the amount rules: minimum, maximum, decimals? Why isn't the amount stated as mandatory, when the note is stated as optional? Zero and negative must be blocked — at UI only, or also in the backend? | Amount is the money itself. A negative amount can reverse the money flow (I saw exactly this in Task 2). | Minimum 0.01, two decimals, zero and negative rejected by both UI and backend. |
| Q5 | Which source account states block a transfer: dormant, frozen, legal hold, closed, insufficient available balance? | The requirement only talks about the beneficiary, but the sender's account can also be in a state that must stop the transfer. | Only active accounts with enough **available** balance can send. |
| Q6 | What if the beneficiary was valid when registered, but is now closed, frozen, newly on a sanctions list, or in a newly sanctioned country — or their bank has stopped operating? Is the check done at execution time? | Sending to a sanctioned account is a regulatory breach for the bank. | Beneficiary status and sanctions are re-checked at execution, not only at registration. |
| Q7 | Purpose note: max length, allowed characters and languages (Arabic and English?), whitespace-only? Is "purpose" mandatory for regulatory reporting? | Some central banks require a purpose code. Free text goes to downstream systems and statements. | Optional, up to 140 characters, letters/numbers/basic punctuation. |

### AC2 — Authentication

| # | Question | Why it matters | My assumption if no answer |
| --- | --- | --- | --- |
| Q8 | How is the OTP delivered: SMS, email, push, in-app, hardware token? | The channel changes what can fail and what we test. | SMS to the registered mobile number. |
| Q9 | OTP length and expiry time? If the customer requests a new OTP, is the old one cancelled immediately? | A delayed SMS + resend is a very common real case. | 6 digits, 5 minutes, a new OTP cancels the old one. |
| Q10 | How many wrong attempts are allowed, and what gets locked: this transfer, the transfer feature for some time, or the whole digital profile? Is there a cooldown on resending OTPs? | Too weak = brute force and SMS-pumping cost. Too strict = a customer locked out of their whole bank account because of an SMS delay. | 3 wrong attempts cancel the transfer and lock transfers for 30 minutes. Resend has a cooldown. |
| Q11 | Is the OTP tied to this customer, this session and this exact transfer (amount + beneficiary)? Can it still be used after the login session expires? | If the OTP is not bound to the transfer details, a changed amount could be confirmed with an old OTP. | Yes, bound to customer, session and transfer details. |

### AC3 — Fees

| # | Question | Why it matters | My assumption if no answer |
| --- | --- | --- | --- |
| Q12 | How is the fee calculated: flat, percentage, by bank (same bank vs other bank), by customer segment? Is it shown to the customer before OTP? | "According to the fee configuration" can't be tested. The customer must see the fee before agreeing. | Flat fee from configuration, shown before OTP. |
| Q13 | Is the fee added on top of the amount, or taken from it? Does the fee count towards the 10,000 / 20,000 limits? | This decides the balance check and every limit boundary test. | Fee is added on top, and does not count towards limits. Balance must cover amount + fee. |

### AC4 — Limits

| # | Question | Why it matters | My assumption if no answer |
| --- | --- | --- | --- |
| Q14 | When does the "day" reset: server midnight, customer's local midnight, or rolling 24 hours? | Defines the boundary test, and customers travelling across time zones. | Calendar day, bank's local time zone. |
| Q15 | Is the limit per customer or per account? Is it shared across channels (app, web, branch)? | Without a shared counter, a customer can bypass the limit by using two channels at the same time. | Per customer, shared across all channels. |
| Q16 | Do failed, cancelled or reversed transfers use up the limit? | A customer could be blocked by money that never left. | Only successful transfers count. |

### AC5 — Successful transfer

| # | Question | Why it matters | My assumption if no answer |
| --- | --- | --- | --- |
| Q17 | Is the debit real-time? Is the credit to the beneficiary real-time or batch (other bank / clearing)? What status does the customer see? | "Success" may mean "accepted", not "delivered". The customer must not be told the money arrived when it didn't. | Debit is real-time. Credit to other banks can be delayed; the screen says "submitted" with the reference. |
| Q18 | If the app closes or the network drops right when the confirmation should load, where does the customer find the reference? | Otherwise they have a lower balance and no proof. | The transfer appears in transaction history with its reference. |
| Q19 | If the SMS service is down, is the confirmation SMS retried? | Without retry, the customer gets no confirmation at all. | SMS is retried; the transfer itself does not depend on the SMS. |

### AC6 — Failure

| # | Question | Why it matters | My assumption if no answer |
| --- | --- | --- | --- |
| Q20 | If the connection drops right after the OTP, and the account was debited but the customer sees an error: does the system return the money automatically, or show the transfer as "pending" until it knows? Who in operations fixes the mismatch? | This is the "money left, but nobody knows where it is" case: the most stressful one for customers and the most expensive for operations. | No money is lost: the transfer ends either completed (visible in history) or fully reversed, never half-done. |
| Q21 | Is a reference generated for failed transfers? Are error messages specific (insufficient funds, limit, beneficiary blocked) or generic? | Customer support needs a reference to investigate. Customers need to know what to fix. | Failed attempts get a reference; messages say the reason without exposing internal details. |

---

## 1.2 Risk assessment

| # | Risk | What could go wrong | Customer | Bank | Us (delivery partner) |
| --- | --- | --- | --- | --- | --- |
| R1 | **Two transfers at the same moment** | Customer sends from the app and the web (or pays by card) at the same second. If the system doesn't lock the balance and share one limit counter, both go through. | Overdraft or negative balance they didn't expect. | Direct money loss, limit controls bypassed, ledger mismatch. | Critical production defect in a money flow, SLA breach. |
| R2 | **Connection drops after OTP — the money left, but the app says it failed** | The network drops right after the OTP. The account is debited, but the app shows an error with no reference number. If the confirmation SMS is delayed or fails, the customer has no proof at all. If the SMS arrives later saying "debited", it contradicts the app's "failed" message, and the customer may send the money again. | Panic, missing money, possible double transfer. | Call-centre spike, manual reconciliation by operations, complaints to the regulator. | Escalations and urgent hotfixes. |
| R3 | **Beneficiary / account state and AML** | Beneficiary was fine at registration but is now closed, frozen or sanctioned, and the system only checked at registration. | Funds stuck or returned days later. | Regulatory fines for sanctions breach, reputational damage. | Liability for a missed business rule. |
| R4 | **OTP security controls** | No retry limit or resend cooldown allows brute force and SMS-pumping. A lockout rule that is too wide locks the customer out of everything because of one SMS delay. | Account drained by fraud, or locked out of their bank. | Fraud compensation, SMS costs, regulator scrutiny. | Security defect traced back to our delivery. |
| R5 | **Fee and limit calculation** | Fee and limit rules are unclear (Q12, Q13), so each team implements its own reading. The fee is not counted in the balance check, or limits are checked on the wrong amount. | Charged a fee they didn't see, or blocked wrongly. | Revenue loss or unfair charges, both lead to complaints. | Rework late in the project. |
| R6 | **Peak load and outages** | Salary days, holidays, or an outage of the core banking or SMS provider. The system has no plan B: no queue to hold transfers until the service is back, and transfers that were half-way through are left unfinished. | Can't send urgent money. | Availability breach, in-flight transactions to reconcile. | SLA default, incident overhead. |

R1 and R2 are my top two. They directly cause lost or duplicated money, and they are the hardest to catch with normal UI testing.

---

## 1.3 Test cases (12)

I used scenarios, not one case per field. With only 12 slots, each case covers a business rule end to end. Basic field checks (mandatory fields, OTP format) go inside these flows instead of taking their own slot.

Common preconditions unless stated: retail customer logged in, active session, active source account, active registered beneficiary in the same currency, fee = 50 (flat), per-transaction limit 10,000, daily limit 20,000, nothing transferred today.

| ID | Title | Covers | Preconditions | Steps | Expected result |
| --- | --- | --- | --- | --- | --- |
| TC01 | Successful transfer end to end, with fee | AC1, AC2, AC3, AC5 | Balance 5,050 | 1. Select source and beneficiary. 2. Enter 5,000 and a note. 3. Check the review screen. 4. Enter correct OTP. | Review screen shows amount 5,000 + fee 50 before OTP. After OTP: balance 0, reference generated, confirmation screen shows amount, fee, beneficiary, reference, date. Same transfer appears in history with the same reference. Confirmation SMS received. Daily used = 5,000. |
| TC02 | Invalid amounts are blocked | AC1 | Balance 10,000 | Try amounts: 0, -100, 100.555 (3 decimals), empty. Also send -100 directly to the API. | Each is rejected before OTP is sent, with a clear message. API rejects with 4xx. No debit, no OTP, no transaction record. |
| TC03 | Balance must cover amount + fee | AC1, AC3 | Balance 5,000 | Transfer 5,000. | Rejected as insufficient funds (5,000 + 50 > 5,000) before OTP. No debit, no fee charged, limit not used. |
| TC04 | Per-transaction limit boundary | AC4 | Balance 25,000 | 1. Transfer 10,000.01. 2. Transfer 10,000. | 10,000.01 is rejected before OTP with a limit message. 10,000 succeeds (fee on top, assumption Q13). |
| TC05 | Daily limit boundary (adding up transfers) | AC4 | Balance 25,000; 15,000 already sent today | 1. Transfer 5,000. 2. Transfer 0.01. | 5,000 succeeds (total exactly 20,000). 0.01 is rejected with a daily-limit message. Remaining limit shown as 0. |
| TC06 | Daily limit resets at the day boundary | AC4 | Limit fully used (20,000) today | 1. Before midnight (bank time): transfer 100. 2. After midnight: transfer 10,000. | Before midnight: rejected. After midnight: succeeds, remaining limit 10,000. |
| TC07 | Wrong OTP — attempts limit and lockout | AC2, AC6 | Balance 5,050 | Start a 1,000 transfer. Enter a wrong OTP 3 times. Then try to start a new transfer. | After the 3rd wrong OTP the transfer is cancelled. No debit, no fee, limit not used. Transfers locked for the agreed time, with a message saying how long. Login and other features still work (lockout scope = transfers only). |
| TC08 | Expired and replaced OTP are rejected | AC2 | Balance 5,050 | 1. Start a transfer; wait past expiry; enter that OTP. 2. Request OTP A, then OTP B; enter OTP A. | Both rejected with clear messages. Customer can request a new OTP. No debit in either case. Using OTP B afterwards completes one transfer only. |
| TC09 | Parallel transfers can't bypass the limit or balance | AC4, R1 | Balance 30,000; 12,000 sent today | From the app and from web, submit two 5,000 transfers at the same time (API level, parallel calls). | Only one succeeds (total 17,000). The other is rejected for the daily limit (it would make 22,000). Balance and limit counter match the one successful transfer. |
| TC10 | Double submit creates only one transfer | AC5, R1 | Balance 10,000 | Tap Confirm twice quickly, or the app re-sends the same confirm request after a slow response. | Exactly one debit, one fee, one reference. Second request returns the same result, not a second transfer. |
| TC11 | Connection drops after OTP — money is never "lost" | AC5, AC6, R2 | Balance 5,050; in the test environment, a way to cut the connection after the bank system debits the account (e.g. a network tool or a fake service) | Confirm with correct OTP; cut the response after the backend debits. Reopen the app. | The transfer ends in one consistent state: either **completed** (visible in history with reference, SMS sent) or **fully reversed** (balance and limit restored). The customer is not told "failed" for money that left. Retrying does not create a second transfer. |
| TC12 | Beneficiary blocked after registration | AC1, AC6, R3 | Beneficiary registered while active, then frozen / flagged in the test core banking system | Start a transfer to this beneficiary and confirm with OTP. | Transfer rejected at execution. No debit, no fee, limit not used. Customer sees a clear message (no internal sanctions detail). Attempt is logged for operations / compliance. |

---

## 1.4 Coverage decisions

**What I chose not to cover, and why**

- **FX / cross-currency and self-transfers** — out of scope by my assumptions (Q2, Q3). If the answer is yes, they each need their own set of cases.
- **Beneficiary registration** — the requirement says "already registered", so it is a separate feature.
- **Field-level UI checks** (note length, OTP format, mandatory fields) — folded into flows or left to developer/unit tests. They are low risk compared to money movement.
- **SMS provider down** — covered partly by TC11 (state must not depend on SMS). A full SMS outage test goes to integration testing.
- **Performance, peak load and DR (R6)** — a real risk, but it needs a separate non-functional plan and environment, not functional cases.
- **Accessibility, localisation, device matrix** — important for a mobile app, but not where money can be lost.

**Automated regression vs manual**

| Automate (API level, runs on every build) | Why |
| --- | --- |
| TC01 (smoke), TC02, TC03, TC04, TC05, TC09, TC10 | Clear inputs and outputs, money rules that must never break, and boundaries that are cheap to repeat. TC09 and TC10 can *only* be done reliably with automation (parallel calls). |
| TC06, TC07, TC08 | Automate at API level using a controllable clock and an OTP test hook in the test environment. |

| Keep manual / exploratory (at least at first) | Why |
| --- | --- |
| TC11, TC12 | Need a way to break the connection on purpose, or special data set-up in core banking. Start manual with the operations team; automate later at API level, once the expected behaviour is agreed. |
| TC01 on real devices | One manual pass on real phones for the full experience: real SMS, app backgrounding, network switching. |

**How I would know it is safe to release**

- All 12 cases pass, with the open questions above answered or the assumptions signed off by the PO.
- No open Critical/High defects on money movement, limits or OTP.
- Money check in test: for every test transfer, the debit, fee and credit match in the ledger, and no transfer is stuck in an unknown state.
- Automated API regression green and stable (not "green after re-runs").
- UAT signed off by the bank's business team, and operations have a runbook for "debited but failed" cases.
