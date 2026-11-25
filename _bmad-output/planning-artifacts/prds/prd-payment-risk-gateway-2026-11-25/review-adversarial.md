---
title: "Adversarial Review — PRD Payment & Fraud Risk Gateway"
reviewed: prd.md, addendum.md (2026-11-25 draft)
date: 2026-11-25
stance: skeptical senior interviewer / payments engineer
---

# Adversarial Review — PRD: Payment & Fraud Risk Gateway

## Verdict

The PRD is well structured and reads like a real spec, but its own headline journey (UJ-2) cannot happen as written, the fraud rules have no defined notion of "history", and several money-correctness guarantees (exactly-once, "eventually one outcome", idempotency) are claimed across stores without saying what happens in the gaps. Fix the two critical items and the history definition before Architecture. Then cut scope, because as written this does not fit one junior in about 3 weeks.

**Counts:** Critical 2 · High 6 · Medium 7 · Low 6 (21 findings)

---

## Critical

### C1. UJ-2 is impossible under the PRD's own rules: Ana cannot afford R$ 2,000
- **Where:** §2.3 UJ-1, UJ-2; FR-6; SM-1.
- **Problem:** In UJ-1 Ana deposits R$ 500 and pays R$ 80, so her Balance is R$ 420. In UJ-2 the attacker pays R$ 2,000 from her account. FR-6 says insufficient Balance returns `422` and nothing is recorded, so no Transaction exists, no event is emitted, and no Risk Analysis or FRAUD_ALERT happens. The flagship fraud demo, and SM-1 (the primary success metric), fail by design.
- **Why it matters:** An interviewer running the Smoke Test will see a `422` where the README promises a reversal. It also shows the journeys were never walked through against the FRs.
- **Fix direction:** Give UJ-2 numbers that work (a larger deposit, or an amount that fits the Balance and is still "unusual"). Have the Smoke Test assert the precondition (Balance ≥ amount) explicitly.

### C2. Whether UJ-2 and the Smoke Test produce FRAUD_ALERT depends on numbers the PRD calls "non-blocking"
- **Where:** FR-11, FR-12, FR-13, FR-14, §3 Cold Start, Q-D, SM-1.
- **Problem:** At the time of UJ-2, Ana has **one** prior Transaction. Unless Cold Start N ≤ 1, Unusual Amount and Atypical Time are both `SKIPPED`, so UJ-2's "impossible travel plus an unusual amount" cannot happen. Only Impossible Travel can fire, so FRAUD_ALERT requires `weight(ImpossibleTravel) ≥ Fraud Threshold`. Atypical Time also depends on the wall-clock hour the Smoke Test runs. A test run at 03:00 and one at 15:00 can disagree, and so can UJ-1 runs (a false FRAUD_ALERT on Ana → Bruno breaks the "COMPLETED" assertion). Q-D calls these numbers "non-blocking because they are configurable", but SM-1 depends on them.
- **Why it matters:** The primary success metric is either nondeterministic or silently relies on an unstated configuration. A flaky demo in front of an interviewer is the worst outcome for a portfolio project.
- **Fix direction:** State the invariant the defaults must satisfy (for example, "Impossible Travel alone reaches the Fraud Threshold", "Cold Start N = X, and the Smoke Test seeds X prior Transactions"). Either make the Smoke Test independent of the time of day or have it set the Rule Configuration explicitly through FR-17 before running. Rewrite UJ-2 so it matches what can actually fire.

---

## High

### H1. "Payer's history" and "previous Transaction" are undefined, so all three rules are ambiguous and the fraud engine can be poisoned
- **Where:** §3 (Unusual Amount, Atypical Time, Impossible Travel, Cold Start), FR-11 to FR-13; addendum "History source for rules".
- **Open questions the PRD must answer:**
  1. **Which statuses count?** Are `FRAUD_ALERT` Transactions in the baseline? If they are, the attacker's Berlin payments become "normal": the next Berlin payment no longer triggers Impossible Travel, and the large fraudulent amounts raise the mean. Ana's next legitimate payment from São Paulo then **fires** Impossible Travel (Berlin → SP), giving a false-positive cascade against the victim. If they are not, say so. Do `PRELIMINARY_APPROVED` (unsettled) Transactions count? Do Deposits count? Does the current Transaction count toward its own baseline?
  2. **Ordering basis:** is "previous" defined by creation time on the Fast Path, or by the order the Slow Path processed or settled them? With retries, backoff, multiple consumers, or DLQ replay, analysis order ≠ creation order. If the worker reads from its own projection (an option the addendum keeps open), the "previous" Transaction may not be visible yet when it is analyzed.
  3. **Snapshot bounding:** a Transaction replayed from the DLQ days later will see newer Transactions as "history" unless history is defined as "created strictly before this Transaction". The same Transaction can then score differently on replay. That conflicts with the dossier being an explanation of *the* decision.
  4. **Cold Start count:** "fewer than N prior Transactions" counted how (all statuses, settled only)?
- **Why it matters:** These are domain semantics, not implementation details. Without them the rules cannot be tested, and the obvious implementation produces the false-positive cascade above, which a payments interviewer will spot quickly.

### H2. FR-9 "eventually exactly one outcome" contradicts FR-18 "stays PRELIMINARY_APPROVED in the DLQ" — money can sit in limbo forever
- **Where:** FR-9, FR-18, §3 Pending Amount, §7.2 (DLQ replay deferred).
- **Problem:** FR-9 promises that every committed Transaction *eventually* gets exactly one outcome. FR-18 sends failures to a DLQ, leaves the Transaction `PRELIMINARY_APPROVED`, and makes replay manual, with the tooling out of scope. The Payer's money is debited and the Payee is never credited, with no bound on how long. Neither the Payer (FR-10 shows only "PRELIMINARY_APPROVED") nor the Admin (no endpoint) can tell a stuck Transaction from a slow one.
- **Why it matters:** "Where is my money?" is the first question in any payments interview. The PRD guarantees something it then explicitly does not deliver.
- **Fix direction:** Weaken FR-9 to say what is actually guaranteed ("eventually analyzed **or** parked in the DLQ, visible, and replayable"), and state the business policy for parked Transactions. Even "manual replay with a documented runbook, and the Smoke Test does not cover it" is acceptable if it is stated. Consider whether a time-boxed default outcome is wanted, and record the decision either way.

### H3. Idempotency has undefined behavior in exactly the cases that cause double charges
- **Where:** FR-5, FR-8, CLAUDE.md rule 3, addendum "Idempotency in Redis".
- **Gaps:**
  1. **The key store is lost or unavailable.** If Redis restarts without persistence or evicts keys, a retry within 24h executes again: a double debit. If Redis is down, does the API fail closed (`503`) or skip the check? The PRD requires the check "before any relational write" but has no durable backstop and states no availability policy.
  2. **Crash between commit and recording the response.** The relational write commits, the process dies before the stored response is saved, and the in-flight marker remains. The client then gets `409` on every retry for up to 24h, or, if the marker expires, the request executes again. FR-8 must say which of these is acceptable.
  3. **Which responses are replayed?** Is a `422 insufficient balance` cached for 24h, so a retry after a Deposit still fails? Is a `500` cached? Is a `400` validation error? Does validation run before or after the key check?
  4. **In-flight duration:** how long can a request hold `409` before the marker is treated as abandoned?
- **Why it matters:** Idempotency is one of the two things the project claims to demonstrate. As written, the tests for FR-8 can pass while the double-charge scenarios remain untested.

### H4. Settlement (PostgreSQL) and the Risk Dossier (MongoDB) cannot both be "exactly one" and "append-only" without a stated failure policy
- **Where:** FR-14, FR-15, FR-16.
- **Problem:** FR-15 requires exactly one Risk Dossier per Transaction, and FR-16 requires one per settled Transaction and forbids updates. These are two stores with no shared transaction. If settlement commits and the worker crashes before writing the dossier, the Transaction is settled without a dossier, and the redelivery is skipped because the status is no longer `PRELIMINARY_APPROVED` (FR-15). If the dossier is written first and settlement then fails, a retry writes a **second** dossier, and append-only rules out fixing it.
- **Why it matters:** FR-15's tested property ("exactly one dossier") is unreachable without an explicit choice, and the PRD treats it as a given.
- **Fix direction:** State which store is authoritative for "was this analyzed", and whether a dossier may exist for an unsettled Transaction, or a settled Transaction may briefly lack one. Make "exactly one" checkable by a key (one dossier per Transaction id) rather than by count.

### H5. FR-7 only covers debit-vs-debit concurrency. Credits and Reversals racing with debits are unspecified and untested
- **Where:** FR-7, FR-14, NFR-6.
- **Problem:** A Payer's Balance is written by Fast Path debits, Deposits, **and** Slow Path Reversals. A Payee's Balance is written by settlement credits while that Payee may be paying someone else. All of these can happen concurrently, possibly from two different services (see the addendum's shared-DB option). FR-7's invariant and test mention only K concurrent payments from one Account. A lost update between a Reversal and a concurrent debit is exactly the bug the design invites.
- **Fix direction:** Extend FR-7 to say "all Balance mutations (debit, Deposit, credit, Reversal) on the same Account". Add a test case: concurrent debits plus a Reversal or Deposit, checked against a conservation invariant (sum of Balances + Pending Amounts = sum of Deposits).

### H6. Scope is too big for one junior in about 3 weeks, and there is no cut line
- **Where:** §7.1, Q-E (end date still open), NFR-5 to NFR-7, SM-1, SM-2.
- **Inventory:** 18 FRs, 9 NFRs, 2 deployable services, 4 datastores plus a broker, an outbox relay, retry and DLQ topology, versioned and cached Rule Configuration, 3 statistical rules, JWT with roles, Testcontainers covering 4 dependencies in CI, a k6 load test with published results, a Smoke Test, a manual QA checklist, and a README/CLAUDE.md rewrite. Several pieces (outbox, exactly-once settlement across two stores, idempotency with failure modes) are each a multi-day job for a junior once done properly.
- **Problem:** There is no priority order inside §7.1, so when time runs out the cut will be made arbitrarily, probably on tests and failure modes, which are the parts that impress.
- **Fix direction:** Set Q-E now. Tag FRs as must or should, with an explicit "cut first" list. Candidates: Atypical Time (the weakest rule and the source of nondeterminism), Rule Configuration *versioning* (keep the endpoint, drop the versions), the k6 publication (NFR-1 / SM-4), and the separate result-queue option in the addendum.

---

## Medium

### M1. `404` for a missing Payee is the wrong status and contradicts FR-10's own anti-enumeration reasoning
- **Where:** FR-6 vs FR-10.
- **Problem:** `POST /transactions` exists, so a `404` suggests the endpoint or resource is missing. The Payee id is a value in the request body, and the usual answer is `422`. Also, FR-10 deliberately returns `404` instead of `403` "so ids cannot be enumerated", while FR-6 lets any Account Holder probe which Account ids exist. Pick one stance on enumeration and apply it to both. If Account ids are sequential, this is a real leak.

### M2. Transaction amount validation is missing
- **Where:** FR-6 (compare FR-5).
- **Problem:** FR-5 rejects amount ≤ 0 and more than 2 decimals for Deposits. FR-6 states nothing about the Transaction amount: zero, negative (a negative transfer is a theft primitive), 3 decimals, or an upper bound. The Fast Path's consequences cannot be tested without these.

### M3. "Same payload" is undefined, and `422` carries three unrelated meanings
- **Where:** FR-6, FR-8.
- **Problem:** A mobile client retrying a payment will often resend *different* Coordinates (the GPS fix moved). Is that the "same payload" or a `422` key reuse? Field order, number formatting (`80` vs `80.00`), and extra fields also need rules. Separately, `422` means insufficient Balance, Payee = Payer, **and** idempotency key reuse with a different payload. NFR-3's error code can tell them apart, but the PRD should list the error codes so clients and tests can assert on them.

### M4. SM-1 claims to validate FR-1–FR-16, but the Smoke Test cannot
- **Where:** SM-1, SM-2.
- **Problem:** A UJ-1/UJ-2/retry script does not exercise FR-7 (concurrency), FR-9 (broker down), FR-15 (redelivery), FR-17 (config change), or most error paths. The metric overstates its coverage. That also breaks NFR-9's own "promise nothing the code does not deliver" principle. Map each FR to the instrument that actually validates it (Smoke Test, integration test, or manual QA).

### M5. Atypical Time has no time zone
- **Where:** FR-12, §3.
- **Problem:** "Time of day" in server UTC, in the Payer's local time (derivable from Coordinates), or in a fixed BRT? A Payer travelling, or a server in UTC, shifts every bucket. Combined with C2 this is the main source of nondeterminism. State the reference clock.

### M6. Impossible Travel edge cases and trust boundary are unstated
- **Where:** FR-13, §3 Coordinates.
- **Problems:** (a) Which timestamp: client-supplied, Fast Path receive time, or analysis time? Analysis time is wrong under backlog. (b) Δt = 0 (two Transactions in the same second, or a burst) gives division by zero, or infinite speed for any non-zero distance. (c) Coordinates are client-supplied, so an attacker who sends São Paulo coordinates defeats the rule. That is fine for an MVP, but the PRD should say so explicitly, because an interviewer will ask.

### M7. Nobody can see Pending Amounts or stuck money
- **Where:** FR-4, FR-10, FR-18, §2.1.
- **Problem:** The Payer's account view shows the reduced Balance with no Pending Amount. The Payee cannot see an incoming Transaction (FR-10 is Payer-only). The Admin has no way to see Transactions parked by FR-18 except the RabbitMQ console. For a product whose pitch is "you get an instant answer and money returned on fraud", the user cannot see where the money is. At minimum, decide whether FR-4 shows a pending total.

---

## Low

### L1. The password rule is "a stated rule" that is never stated
- **Where:** FR-1. As written, the `400` consequence cannot be tested. State the rule (length, classes) or drop "weak password".

### L2. FR-7's formula carries hidden test preconditions
- **Where:** FR-7. "Exactly ⌊B/A⌋ succeed, the rest `422`" assumes distinct Idempotency Keys (otherwise some get `409` or a replay), no concurrent Deposit or Reversal, and a locking strategy that never surfaces an exhausted-retry error as `409` or `500`. State that any outcome other than `201` or `422` fails the test.

### L3. The NFR-1 load profile is underspecified
- **Where:** NFR-1. One hot Account (lock contention) or many Accounts? Is warm-up excluded? Is the token pre-issued? These change p95 by an order of magnitude, so the published number can't be reproduced.

### L4. Deposits print money, without limits or analysis
- **Where:** FR-5, §6. A stolen session can deposit and then pay, and there is no cap. This is acceptable for a demo but belongs in §6 Non-Goals so it reads as deliberate.

### L5. Admin behavior on Account Holder endpoints is unspecified
- **Where:** FR-3, FR-4, FR-10. FR-3 says Admin gets `403` on Transactions and Deposits (create), but not on FR-4 (no Account) or FR-10 (query). Should these be `403` or `404`?

### L6. The Rule Configuration can be set so that fraud is impossible, and concurrent updates can be lost
- **Where:** FR-17. Validation allows a Fraud Threshold above the sum of all weights, which silently disables detection. A full-replace update from two Admin sessions loses one change, and nothing says which version wins. Either add the threshold ≤ sum of weights check or state that it is allowed. One Admin makes the concurrency case minor.

---

## Things that hold up (for balance)

- Separating the Fast Path and Slow Path, with FR-9's broker-down consequence, is the right core idea, and its consequence can be tested.
- FR-15's "no longer PRELIMINARY_APPROVED → do not re-settle" is the correct guard, once H4 is resolved.
- The status codes for idempotency (`409` in flight, `422` payload mismatch) match the IETF Idempotency-Key draft.
- The counter-metrics (SM-C1 to SM-C3) and NFR-9 (README ⊆ code) are unusually mature for a portfolio PRD. Keep them.
