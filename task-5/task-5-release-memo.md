# Task 5 — Release Memo: International Transfer Journey

**To:** Delivery Manager  **From:** QA Lead  **Re:** "Is QA green?" — go-live in 3 working days

## 5.1 Recommendation

**QA is not green today, and I can't confirm it in writing.** My recommendation is **No-Go for the current date, with a conditional Go** if all of the following are true by the end of Day 2 (cut-off agreed with the client):

1. **UAT signed off** by the client's business team, covering all international transfer scripts.
2. **All regression failures explained.** Each of the ~30 "flaky" tests is either fixed and passing, or shown to be a test problem with the flow passed manually. Any real failure in transfer, FX, limits or fees blocks the release.
3. **FX integration tested beyond the happy path:** rate unavailable / timeout, stale or expired rate, rounding, amount and limit boundaries, and the rate shown = the rate applied.
4. **The two KYC criticals are proven not to affect transfers.** Dev lead confirms which profile-service calls the transfer journey uses, and we run the transfer regression against those calls. If they are affected, they must be fixed before go-live.
5. A tested **rollback / feature-flag-off plan**, and operations know how to handle a "debited but failed" transfer.

If any condition is not met, we move the date. A delay costs us a difficult conversation with marketing; a broken international transfer costs customers' money and the bank's reputation with the regulator.

**Why:** 68% is not a pass rate I can sign for in a money flow. "Just flaky" is an assumption nobody has checked. UAT is half-done. The highest-risk new component (FX) has had one round of happy-path testing, and it handles the price customers pay.

## 5.2 Next 72 hours (priority order)

1. **Day 1 AM — Triage the regression failures** (2 senior QAs + 1 developer). Sort every failure into: real defect / environment / test issue. Real defects in scope are logged and fixed first.
2. **Day 1–2 — FX negative testing** (most experienced QA), on staging with the real upstream.
3. **Day 1–3 — Support UAT.** I sit with the client's business team daily to unblock their remaining scripts, and we fix their defects as priority.
4. **Day 1 — Profile service impact** check with the dev lead, then a targeted transfer + profile regression.
5. **The two newer QAs** run the manual checks for quarantined tests, using scripts written by the senior QAs, and pair with them on FX — not alone on the riskiest area.
6. **Day 3 — Go/No-Go meeting** with evidence against the five conditions.

**What I drop:** writing new automation, low/medium defects outside the release, polishing reports, and any regression outside the transfer, FX, profile and payment paths.

## 5.3 Communication

**(a) Delivery manager — in writing, today.** Honest and full: the actual numbers, what is unknown, the five conditions, the plan, and what I need (one developer on triage, a decision owner for Day 3). I won't write "QA is green". I will write exactly what is tested, what isn't, and the risk.

**(b) Client — with the delivery manager, Day 1.** Focused on their business: "Development is complete. We are finishing UAT with your team and the final checks on the FX rate integration. We will confirm go-live on Day 3, and we recommend marketing doesn't fix a launch message before then."

**Where they differ:** the level of internal detail — test triage, team experience, our process gaps stay internal.
**Where they must not differ:** the facts. The same status, the same risks that affect the client (FX, profile service, UAT), the same conditions and the same decision date. We never tell the client "on track" while telling ourselves "at risk".

## 5.4 What I would change, and how to measure it

**Practices:**
- **Flaky-test quarantine with an owner.** A test that fails without a code change is moved out of the blocking suite within a day, gets a ticket with an owner, and is fixed or deleted within the sprint. The blocking suite must be 100% green — "68% and probably fine" stops being possible.
- **Integrations land early.** An external integration (like FX) must be on staging, with negative tests, at least one sprint before go-live — part of our Definition of Done, not left to the last week.
- **UAT runs during the sprints, not at the end.** The client's team tests each feature when it's done, with QA supporting them, so the final UAT is a confirmation, not a first look.
- **Shared-service changes trigger the regression of every journey that uses the service** (here: profile service → onboarding and transfers).

**Metrics shared with the client (weekly):**
1. **Defects found after QA** (in UAT or production), by severity — tells the client how well our testing catches issues.
2. **Release readiness by risk area** — for each high-risk flow (transfer, FX, limits, fees): tests passed, open defects, UAT status.
3. **Blocking suite health** — pass rate of the stable suite, plus number of quarantined tests and their age.

**What I would not measure:** the number of test cases written or executed, or test coverage % on its own. They are easy to make look good without lowering risk — a bigger suite is what gave us 30 unexplained failures.
