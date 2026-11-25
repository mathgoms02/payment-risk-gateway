# PRD Quality Review — Payment & Fraud Risk Gateway

## Overall verdict
This is a strong PRD for a solo portfolio project at internal-tool rigor. It has a clear thesis (Fast Path and Slow Path are separated), a tight glossary, mostly testable FRs, honest non-goals and counter-metrics, and an addendum that hands real decisions to Architecture. What's at risk sits in the Slow Path semantics. "Payer history" is never defined, so the three rules, Cold Start, and UJ-2 itself are under-specified. FR-9's promise of "exactly one Risk Analysis outcome" contradicts the DLQ behaviour in FR-18. And the primary success metric (SM-1) depends on default rule numbers that §9 calls "non-blocking". Fix those three and the PRD is ready for Architecture.

## Decision-readiness — strong
Decisions are stated as decisions. Polling is chosen over webhooks (§7.2, addendum C). The weighted Risk Score is chosen over "any single rule ⇒ fraud". Rules are history-based with a Cold Start instead of fixed thresholds. Login issues a JWT, which settles kickoff Q4. Rejected alternatives are written down with their reasons (addendum C). The open questions really are open: Q-B and Q-C each have an owner and are tied to the FR they must satisfy ("Must satisfy FR-9", "Must satisfy FR-7"). That is exactly how to hand a decision to Architecture. The weak spot is one open question labelled non-blocking when it actually is blocking.

### Findings
- **[high]** Q-D is called "non-blocking" but the primary success metric depends on it (§9 Q-D, §2.3 UJ-2, §8 SM-1). In UJ-2 the fraud is caught because "impossible travel plus an unusual amount" fires. But Ana has only one prior Transaction (UJ-1), so Unusual Amount is "skipped during Cold Start" (FR-11) for any N > 1. The Smoke Test on a fresh clone (seed "off by default", NFR-5) then passes only if Impossible Travel's weight alone reaches the Fraud Threshold, or if the script reconfigures the rules first. Neither is stated. *Fix:* state the constraint the defaults must meet ("with the defaults, Impossible Travel alone ≥ Fraud Threshold"), or state that the Smoke Test sets up its own Rule Configuration and history through the API. Also rewrite UJ-2 so it matches what the defaults will actually fire.

## Substance over theater — strong
Every persona does real work. The Interviewer drives UJ-3, SM-1, and FR-18's framing. The Admin drives FR-17. The Builder JTBD explains NFR-7 and NFR-9. The NFRs carry real thresholds (NFR-1: p95 < 100 ms at 50 req/s for 60 s, measured and published) instead of adjectives. The Vision is specific to this product. Only one Vision claim reads as furniture.

### Findings
- **[medium]** The Vision makes resilience claims that no FR or NFR backs (§1, paragraph 2). "A broken fraud worker never blocks checkout" and "a traffic spike on checkout never starves the risk analysis" are the thesis in one sentence. Yet FR-9 only covers the *broker* being down, nothing covers the *worker* being down, and nothing tests starvation or independent scaling. *Fix:* add a consequence to FR-9 or FR-18 ("with the Fraud Worker stopped, `POST /transactions` still returns `201`; when the worker restarts, pending events are processed"), and drop or soften the starvation/scaling claim.

## Strategic coherence — strong
The thesis holds across the document. The fast/slow split shapes §4.3 versus §4.5, the latency NFR, the polling decision, and the resilience feature. The success metrics validate the thesis (a working end-to-end demo plus measured Fast Path latency), not activity counts. The counter-metrics are well chosen (SM-C1 coverage %, SM-C2 rule count, SM-C3 latency below target), and each one names the metric it balances. The MVP is a coherent "experience/demo" scope for a portfolio piece.

### Findings
- **[medium]** SM-1 claims more coverage than the Smoke Test delivers (§8 SM-1). It says "Validates FR-1–FR-16", but the Smoke Test runs only UJ-1, UJ-2, and one idempotent retry. It does not exercise FR-3 (Admin bootstrap), FR-7 (concurrency), FR-9 (broker down), FR-12 (Atypical Time), FR-15 (redelivery), or most of FR-8's branches. Those are covered by NFR-6 integration tests or SM-2. *Fix:* list the FRs the Smoke Test actually covers (FR-1, 2, 4–6, 8 partial, 10, 13, 14, 16) and leave the rest to SM-2 and NFR-6.

## Done-ness clarity — adequate
Most FRs read like acceptance tests, with explicit status codes, invariants, and a concurrency formula (FR-7's "exactly ⌊B/A⌋ succeed" is excellent). Idempotency (FR-8) covers replay, payload mismatch, in-flight requests, and expiry. The weakness is concentrated in the Slow Path. The rule definitions and the meaning of "history" are left loose, and there is one outright contradiction. Story creation will stall there first.

### Findings
- **[high]** "Payer history" is undefined, so the rules and Cold Start are not testable (§3 Cold Start, FR-11, FR-12, FR-13). Nothing says which prior Transactions count: all of them, `COMPLETED` only, or `FRAUD_ALERT` too, and whether still-`PRELIMINARY_APPROVED` ones count. It also isn't said whether "previous Transaction" in Impossible Travel means the previous one by creation time or by settlement. In UJ-2 this matters right away. If the fraudulent Berlin Transaction counts as history, Ana's next real payment from São Paulo fires Impossible Travel against the attacker's Coordinates. Event processing order is also left unstated, which matters once two Transactions are in flight together. *Fix:* add a Glossary entry "Payer History" (for example: Transactions by the Payer created before this one, excluding `FRAUD_ALERT`), and point FR-11/12/13 and Cold Start at it.
- **[high]** FR-9 contradicts FR-18 on outcomes, and DLQ'd money has no stated end state (FR-9, FR-18). FR-9 says every Transaction "eventually produces exactly one Risk Analysis outcome". FR-18 says that after the retries the Transaction "remains `PRELIMINARY_APPROVED`" in the DLQ, and replay is manual and out of scope. As written, the Payer is debited while the Payee is never credited, possibly forever. *Fix:* narrow FR-9 to "exactly one event delivered (at least once) / at most one outcome". Then state openly in FR-18 that a DLQ'd Transaction keeps its Pending Amount until someone replays it by hand, and record this as an accepted MVP limitation.
- **[medium]** FR-6 has no amount validation (FR-6). FR-5 says "Amount ≤ 0 or with more than 2 decimal places returns `400`" for Deposits, but FR-6 says nothing about the amount of a Transaction. A zero or negative transfer could be written as a "valid" story. *Fix:* copy FR-5's amount rule into FR-6.
- **[medium]** The rule definitions are only examples (FR-11, FR-12). "The baseline definition (for example, a multiple of the mean of the last M Transactions) is configurable" and "The pattern definition (for example, hour-of-day buckets…)" leave the engineer to invent the algorithm. FR-12 also has no concrete firing example, while FR-13 has one (Brazil to Germany). *Fix:* commit to one definition per rule (the examples are fine as the definition) and name their parameters in FR-17, so that "configurable" means changing parameters, not swapping algorithms.
- **[low]** FR-1 refers to "weak password per a stated rule", but no rule is stated. *Fix:* state it (for example, a minimum of 8 characters).
- **[low]** FR-8 doesn't say whether failed outcomes are replayed (FR-8). It isn't clear whether a `422` (insufficient Balance) or a `5xx` is stored under the key and replayed, or whether a retry is allowed to run the request again. *Fix:* add one line, such as "4xx business outcomes are stored; 5xx/unexpected failures release the key".
- **[low]** FR-12 doesn't say Cold Start skips are recorded as `SKIPPED` the way FR-11 does. FR-16 implies it, so the gap is only in the wording.

## Scope honesty — strong
§6 Non-Goals does real work (no PIX, no MFA, no refunds, no Kafka, no UI), and §7.2 gives a reason for each deferral. Nine `[ASSUMPTION]` tags and four Open Questions, with no `[NOTE FOR PM]`, is a fine density for a solo, low-stakes build. The only silent omission is the DLQ stuck-funds state (see Done-ness).

### Findings
- **[low]** Deposits have no upper bound and no rationale (FR-5). Unlimited, risk-free money creation is fine for a demo. But an interviewer may ask why Deposits skip Risk Analysis, and the PRD doesn't say. *Fix:* add a one-line `[NON-GOAL for MVP]` or rationale ("Deposits simulate an external funding source; not a fraud vector in scope").

## Downstream usability — adequate
The Glossary is strong and used consistently (Payer, Payee, Pending Amount, Reversal, and Fast/Slow Path all appear verbatim). The FR, NFR, and SM IDs are contiguous, and each feature says which UJ or JTBD it realizes. Addendum A already lists the Architecture decisions (outbox, locking, settlement write path, history source, rule-config storage). One cross-store consistency concern is missing from it.

### Findings
- **[medium]** Cross-store atomicity of settlement plus dossier is not handed to Architecture (FR-14, FR-15, FR-16, addendum A). FR-15 requires "exactly … one Risk Dossier" per settlement, but settlement happens in PostgreSQL and the Risk Dossier lives in MongoDB. That is a second dual-write problem, the same in kind as Q-B, and the addendum doesn't mention it. *Fix:* add an addendum A bullet that says how the dossier write is made idempotent and consistent with settlement (for example, the Transaction id as the dossier `_id`, written before or after the PostgreSQL commit, with an upsert-insert).
- **[low]** The two services, "Payment API" and "Fraud Worker", are named in NFR-5 and §7.1, but the Glossary doesn't define them, and §4.5 calls the second one "a separate worker". *Fix:* add both to §3.

## Shape fit — strong
Capability-spec shape fits an API-only tool with few roles. There are three UJs, each with a named protagonist (Ana, Ana/attacker, the Interviewer), and each does real work: UJ-3 doubles as the Smoke Test spec. The length (about 320 lines) fits the 5–8 page target. Nothing is over-formalized. Treating the portfolio audience as a stakeholder (Interviewer and Builder JTBDs, NFR-9) is an honest fit for the real goal.

## Mechanical notes
- **Open Question IDs:** they start at Q-B, and there is no Q-A. Q-B and Q-C map to kickoff Q2 and Q3, so the letters look like a leftover. Renumber them, or say that Q-A (kickoff Q1) was settled by §7.2 and NFR-9.
- **Assumptions Index roundtrip:** clean. All nine inline `[ASSUMPTION]` tags (§3 Admin, FR-3, FR-10, FR-11, FR-12, FR-16, FR-18, NFR-1, §6) appear in §10, and every §10 entry appears inline.
- **Status code overloading:** `422` means insufficient Balance (FR-6), Payee = Payer (FR-6), and idempotency payload mismatch (FR-8). This is acceptable only because NFR-3 includes an error code. The error-code values should be listed in Architecture so the Smoke Test can tell them apart.
- **Unspecified FR-2 token expiry duration:** fine to leave to Architecture, but it is worth one `[ASSUMPTION]`.
- **Terminology:** the Vision and the UJ titles say "payment", and the body says "Transaction". The difference is harmless, but the Glossary could note "payment = Transaction".
- **Required sections:** all present for the agreed stakes: Vision, users/JTBD, UJs, Glossary, FRs, NFRs, Non-Goals, Scope, SMs with counter-metrics, Open Questions, and the Assumptions Index.
