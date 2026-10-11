# Task 2 — Exploratory Testing & Bug Reporting

**Application:** ParaBank — https://parabank.parasoft.com/parabank/index.htm (primary site worked; I didn't need the backup)
**Date tested:** 09-Oct-2026
**Scope:** functional defects only. No security, load or penetration testing.

ParaBank is a shared public demo and its data changes. On one run new users started with $10,000, on another with $515.50. The steps below work for any new user — only the numbers change.

Both bugs below were reproduced on **two different users**: once through the screens (screenshots in `evidence/`), and once by calling the same backend endpoints the screens use.

---

## 2.1 Bug reports

### BUG-01 — Transfer Funds lets a customer send more money than the account has (balance goes to –$400)

| | |
| --- | --- |
| **Feature** | Transfer Funds |
| **Severity** | **High** — breaks the most basic money rule: you can't send money you don't have. Each case is a loan the bank never approved. |
| **Priority** | **High** — Transfer is a main feature every customer uses; it must be fixed before release. |

**Environment**
- URL: https://parabank.parasoft.com/parabank/index.htm
- Desktop browser (screens) + direct calls to the same endpoint (`POST /parabank/services_proxy/bank/transfer`).

**Preconditions**
- A new user, logged in, with two accounts: the default CHECKING, and a SAVINGS opened from **Open New Account** (it moves the minimum $100 from CHECKING).
- Screenshot run: user `realrec776731` — CHECKING `17784` = $9,900.00, SAVINGS `17895` = $100.00.

**Steps to reproduce**
1. Register a new user and log in. *(evidence 00, 01)*
2. Open **Open New Account**, choose SAVINGS, fund it from the CHECKING account, click **Open New Account**. *(02, 03)*
3. Open **Accounts Overview** and note the balances: CHECKING $9,900.00, SAVINGS $100.00. *(04)*
4. Open **Transfer Funds**. Amount: `500`. From: the SAVINGS account (`17895`). To: the CHECKING account (`17784`). *(bug01-1)*
5. Click **Transfer**.
6. Open **Accounts Overview** again.

**Actual result**
- Step 5: "Transfer Complete! $500.00 has been transferred from account #17895 to account #17784." *(bug01-2)*
- Step 6: SAVINGS **$100.00 → –$400.00**, CHECKING $9,900.00 → $10,400.00. *(bug01-3)*

**Expected result**
- The transfer is stopped with an "insufficient funds" message. No balance changes.
- Where my expectation comes from:
  1. **Banking rule:** money out can't be more than the available balance, unless the customer has an agreed overdraft. ParaBank offers none.
  2. **ParaBank itself:** the overview shows SAVINGS with *Available Amount $0.00* next to the –$400 balance. So the app knows a negative balance means "nothing available" — but Transfer still let the customer create one.

**Evidence**
- Screenshots: `evidence/04-overview-before.png`, `bug01-1-transfer-form-500.png`, `bug01-2-transfer-complete.png`, `bug01-3-overview-savings-negative.png`
- Second user, via the endpoint (user `final210932`, SAVINGS `14676`, CHECKING `14565`):
  - `POST /parabank/services_proxy/bank/transfer?fromAccountId=14676&toAccountId=14565&amount=500`
  - Response `200`: `Successfully transferred $500 from account #14676 to account #14565`
  - Before `{"14565": 9900.0, "14676": 100.0}` → after `{"14565": 10400.0, "14676": -400.0}`

```bash
# After logging in (session cookie saved in "jar"); use your own account numbers
curl -c jar -b jar -s -X POST "https://parabank.parasoft.com/parabank/services_proxy/bank/transfer?fromAccountId=17895&toAccountId=17784&amount=500"
```

---

### BUG-02 — Bill Pay accepts a negative amount: paying –$50 adds $50 to the payer's account

| | |
| --- | --- |
| **Feature** | Bill Pay |
| **Severity** | **High** — any customer can raise their own balance by "paying" negative bills. Money is created from nothing. |
| **Priority** | **High** — Bill Pay sends money out of the bank; this is a direct fraud risk and breaks the books. |

**Environment**
- Same as BUG-01. Endpoint: `POST /parabank/services_proxy/bank/billpay`.

**Preconditions**
- Logged-in user with money in CHECKING.
- Screenshot run: same user `realrec776731`, CHECKING `17784` = $10,400.00 before this test.

**Test data — payee**
| Field | Value |
| --- | --- |
| Payee Name | Electric Co |
| Address / City / State / Zip | 1 Power St / Dublin / D / 12345 |
| Phone # | 123456 |
| Account # and Verify Account # | 12345 |

**Steps to reproduce**
1. Log in and open **Accounts Overview**. Note the CHECKING balance ($10,400.00). *(bug01-3)*
2. Open **Bill Pay** and fill the payee fields with the test data above.
3. Amount: `-50`. From account: CHECKING (`17784`). *(bug02-1)*
4. Click **Send Payment**.
5. Open **Accounts Overview** again.

**Actual result**
- Step 4: "Bill Payment to Electric Co in the amount of $-50.00 from account 17784 was successful." *(bug02-2)*
- Step 5: CHECKING **$10,400.00 → $10,450.00**. The payer got money instead of paying. *(bug02-3)*
- Amount `0` also shows success and records a $0.00 payment.

**Expected result**
- Any amount of 0 or less is refused with "Please enter a valid amount". No payment is recorded.
- Where my expectation comes from:
  1. **ParaBank itself:** the Bill Pay form already has the error "Please enter a valid amount", so the app means to check the amount — but it only checks that it's a number.
  2. **Banking rule:** a bill payment takes money out. It must always be more than zero. A negative payment turning into money in is never valid.

**Evidence**
- Screenshots: `evidence/bug02-1-billpay-form-minus50.png`, `bug02-2-billpay-complete.png`, `bug02-3-overview-checking-up-50.png`
- Second user, via the endpoint (user `final210932`, CHECKING `14565`):
  - `POST /parabank/services_proxy/bank/billpay?accountId=14565&amount=-50` (payee JSON below)
  - Response `200`: `{"payeeName":"Electric Co","amount":-50,"accountId":14565}`
  - Before `{"14565": 10400.0}` → after `{"14565": 10450.0}`

```bash
curl -c jar -b jar -s -X POST "https://parabank.parasoft.com/parabank/services_proxy/bank/billpay?accountId=17784&amount=-50" \
  -H "Content-Type: application/json" \
  -d '{"name":"Electric Co","address":{"street":"1 Power St","city":"Dublin","state":"D","zipCode":"12345"},"phoneNumber":"123456","accountNumber":12345}'
```

**Why these are two different bugs:** BUG-01 is a missing **"do you have enough money?"** check on Transfer — the amount is normal, the balance is too low. BUG-02 is a missing **"is the amount more than zero?"** check on Bill Pay — it happens even when the account has plenty of money. Different feature, different rule, different fix.

---

## 2.2 Approach

**Where I started and why.** I registered a new user and opened Accounts Overview first, to know the starting balances. Every finding after that is a "before vs after" on those numbers. Then I went straight to the features that move money — Transfer Funds, Bill Pay, Request Loan — because that's where a bug costs real money. Contact info and search came after.

**What I found by hand first.** Very quickly I saw there's almost no checking anywhere. Transfer Funds accepts the same account on both sides, zero, negative, and more than the balance. Bill Pay, Request Loan and Update Contact Info have the same gap. That pattern told me where to dig. The two bugs above are the clearest cases of it.

**How I confirmed.** For each money form I tried what a bank must refuse: zero, negative, more than the balance, same account on both sides. I also checked the calls the screens send to the backend. Each page is a thin layer over `services_proxy/bank/...`, and the only checks are "not empty" and "is a number" in the browser — nothing in the backend. So I confirmed each bug at both levels: what the customer sees, and what the backend accepts and saves.

**Other findings I didn't report as main bugs** (same family as the two above, or the expected result isn't clear):
- Transfer also accepts negative and zero amounts, and transfers to the same account — same "amount must be more than zero" family as BUG-02.
- Bill Pay also allows paying more than the balance (a $15,000 payment from a $10,000 account went to –$5,000) — same "enough money?" family as BUG-01, on a different feature.
- Request Loan approved two $1,000 loans in a row with $0 down payment, and the money was usable straight away (`evidence/other-findings/`). I didn't make it a main bug because ParaBank doesn't say what its lending rules are, so my "expected result" would be my opinion. I'd ask the PO first.
- Transfer to an account that doesn't exist (`999999`) says "Successfully transferred", but nothing moves — a false success message.
- The Administration page opens without an admin login. I noted it and didn't go further (security testing was out of scope).

**If I had two more hours:**
- **Find Transactions** — date and amount filters, and an account that doesn't exist (I saw an HTML error page instead of a clean error message).
- **Request Loan** — find the real approval rule (amount vs balance vs down payment) and test around it.
- **Update Contact Info** — empty and space-only values, and whether changes are really saved.
- **Open New Account** — the $100 minimum deposit when the source account is almost empty.
- Re-test both bugs in a second browser, to rule out effects from other people using the same demo.
