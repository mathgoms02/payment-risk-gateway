---
title: "PRD — Payment & Fraud Risk Gateway"
status: final
created: 2026-11-25
updated: 2026-11-25
---

# PRD: Payment & Fraud Risk Gateway

## 0. Document Purpose

This PRD defines **what** the Payment & Fraud Risk Gateway MVP must do. It is the input for the next BMad steps: Architecture, Test Design, and Epics & Stories. It builds on `docs/kickoff.md`, `README.md`, `CLAUDE.md`, and the target job posting (Junior Backend Software Engineer). It settles kickoff §5 open questions Q1 (§7), Q4 (FR-2), and Q5 (NFR-9), and states the requirements behind Q2 (FR-9) and Q3 (FR-7) without choosing their mechanisms; those remain open in §9. Domain terms are defined once in §3 and used verbatim everywhere else. Features (§4) nest globally numbered FRs. Cross-cutting NFRs live in §5. Inferred items carry `[ASSUMPTION]` tags and are indexed in §10. Implementation choices and rejected alternatives live in `addendum.md`.

## 1. Vision

Payment & Fraud Risk Gateway is a payment backend that never makes the buyer wait for fraud analysis. A payment request is accepted or rejected in milliseconds, based only on what must be checked synchronously: authentication, idempotency, and available balance. The deeper Risk Analysis (unusual amounts, atypical hours, mismatched geolocation) runs asynchronously in a separate worker. When that analysis flags a transaction as fraud, the system marks it and returns the money to the payer's account. Every analysis, fraudulent or not, ends up in an auditable Risk Dossier.

The design separates the Fast Path from the Slow Path, so each can fail, scale, and evolve independently. A broken fraud worker never blocks checkout, and a traffic spike on checkout never starves the Risk Analysis.

## 2. Target User

The product is an API with no UI. It has two human roles (**Account Holder**, **Admin**) and one evaluation audience (**Interviewer**). This is a portfolio project, so the builder is also a stakeholder.

### 2.1 Jobs To Be Done

- **Account Holder:** send money to another account and get an immediate answer, without being slowed down by fraud checks, and be protected (money returned) if the payment turns out to be fraudulent.
- **Admin ("bank owner"):** tune how aggressive fraud detection is without redeploying the system.
- **Interviewer:** within minutes, verify that the system works end-to-end and that the candidate understands the trade-offs behind it.
- **Builder (portfolio):** demonstrate Java/Spring, SQL + NoSQL, messaging, CI/CD, and an AI-assisted process (BMad artifacts) that match the target job posting.

### 2.2 Non-Users (v1)

- Merchants integrating through webhooks (webhooks are deferred, see §7.2).
- End users on a web or mobile UI (an Angular UI is a far-future idea, not v1).

### 2.3 Key User Journeys

- **UJ-1. Ana pays Bruno and gets an instant answer.** Ana registers, logs in, deposits R$ 500, and sends R$ 80 to Bruno's account from her phone in São Paulo. She gets `201 PRELIMINARY_APPROVED` immediately. Seconds later, a status query shows `COMPLETED`, and Bruno's balance has gone up by R$ 80.
- **UJ-2. A stolen session is caught after the fact.** Ana's credentials are used minutes later to pay R$ 400 from coordinates in Berlin. The payment is accepted on the Fast Path, but the Risk Analysis flags impossible travel (São Paulo to Berlin in minutes). The transaction becomes `FRAUD_ALERT`, R$ 400 goes back to Ana's account, Bruno is never credited, and a Risk Dossier records why.
- **UJ-3. The Interviewer runs the Smoke Test.** On a fresh clone, the Interviewer runs `docker compose up` and then the Smoke Test script. The script exercises UJ-1, UJ-2, and an idempotent retry, and prints a pass/fail summary.

## 3. Glossary

- **User** — A registered identity that can log in. Has exactly one role: Account Holder or Admin.
- **Account Holder** — A User who owns exactly one Account.
- **Admin** — A User who manages the Rule Configuration ("bank owner"). Owns no Account. `[ASSUMPTION]`
- **Account** — Holds a Balance. Belongs to exactly one Account Holder.
- **Balance** — The amount available in an Account. Never negative.
- **Deposit** — An Account Holder adding money to their own Account. Not subject to Risk Analysis.
- **Transaction** — A transfer of an amount from the Payer's Account to the Payee's Account.
- **Payer** — The Account Holder who creates a Transaction. Always the authenticated User.
- **Payee** — The Account receiving the Transaction amount. Must exist and differ from the Payer's Account.
- **Transaction Status** — One of `PRELIMINARY_APPROVED`, `COMPLETED`, `FRAUD_ALERT`. Transitions only from `PRELIMINARY_APPROVED` to one of the other two, and at most once.
- **Pending Amount** — The amount debited from the Payer and not yet credited to the Payee, while the Transaction is `PRELIMINARY_APPROVED`.
- **Reversal** — Returning a `FRAUD_ALERT` Transaction's amount to the Payer's Account. Happens exactly once per Transaction.
- **Idempotency Key** — A client-supplied value in the `Idempotency-Key` header. It makes a repeated write request return the original result instead of executing twice.
- **Fast Path** — The synchronous handling of a Transaction request, up to the `201` response.
- **Slow Path** — The asynchronous Risk Analysis that follows the Fast Path.
- **TransactionCreatedEvent** — The message the Fast Path emits for each new Transaction. It triggers the Slow Path.
- **Coordinates** — Latitude and longitude of the Payer at request time, sent by the client.
- **Risk Rule** — A check that, when it fires, adds points to the Risk Score. There are three in the MVP: Unusual Amount, Atypical Time, Impossible Travel.
- **Unusual Amount** — A Risk Rule that fires when the amount deviates strongly from the Payer's own Transaction history.
- **Atypical Time** — A Risk Rule that fires when the time of day deviates from the Payer's own Transaction pattern.
- **Impossible Travel** — A Risk Rule that fires when the distance between the Coordinates of the Previous Transaction and the current one cannot be covered in the time between them.
- **Payer History** — The Payer's earlier `COMPLETED` Transactions. `FRAUD_ALERT` and still-pending Transactions never count, so fraud cannot become "normal" behavior.
- **Previous Transaction** — For Impossible Travel only: the Payer's most recent Transaction created before the current one that is not `FRAUD_ALERT` (pending ones count, to catch attacks fired in quick succession).
- **Cold Start** — The state of a Payer whose Payer History has fewer than N Transactions. History-based Risk Rules are skipped during it.
- **Risk Score** — The sum of points from the Risk Rules that fired.
- **Fraud Threshold** — The Risk Score at or above which a Transaction becomes `FRAUD_ALERT`.
- **Rule Configuration** — The weights, Fraud Threshold, N, and rule parameters in effect. Changed by an Admin at runtime.
- **Risk Analysis** — The Slow Path evaluation of one Transaction, producing a Risk Score and a Transaction Status.
- **Risk Dossier** — The auditable record of one Risk Analysis.
- **Dead Letter Queue (DLQ)** — The holding place for TransactionCreatedEvents whose Risk Analysis failed after all retries.
- **Smoke Test** — A script that runs UJ-1 to UJ-3 against a running stack and reports pass/fail.

## 4. Features

### 4.1 Identity & Access

**Description:** A person registers as an Account Holder, which creates their User and their Account in one step, and logs in to receive a JWT. Every other endpoint requires that JWT, and authorization is role-based, which keeps a future UI possible without changing the contract. One Admin exists from first startup. Realizes UJ-1, UJ-3.

#### FR-1: Register an Account Holder

A visitor can register with a username and password, which creates a User with the Account Holder role and one Account with Balance 0.

- Duplicate username returns `409`.
- Invalid payload (blank fields, weak password per a stated rule) returns `400` with field-level errors.
- Passwords are never stored or logged in plain text.

#### FR-2: Log in

A User can exchange valid credentials for a signed JWT carrying their identity and role.

- Valid credentials return `200` with a token and its expiry.
- Invalid credentials return `401` without revealing whether the username exists.
- Requests with a missing, malformed, or expired token to protected endpoints return `401`. A valid token without the required role returns `403`.

#### FR-3: Admin bootstrap

An Admin User exists after first startup, with credentials supplied through environment configuration, not hard-coded in the repository. `[ASSUMPTION]`

- Starting the stack with the admin variables set allows that Admin to log in.
- An Admin cannot create Transactions or Deposits (`403`).

### 4.2 Accounts & Deposits

**Description:** An Account Holder can see their own Account and add money to it. Deposits are critical writes and follow the same idempotency rules as Transactions, but they skip Risk Analysis. Realizes UJ-1.

#### FR-4: View own Account

An Account Holder can retrieve their Account's identifier and current Balance.

- The response reflects every committed Deposit, Transaction debit, credit, and Reversal.
- No endpoint exposes another Account Holder's Balance.

#### FR-5: Deposit into own Account

An Account Holder can deposit a positive amount into their own Account, with an Idempotency Key.

- A successful Deposit increases Balance by exactly the amount and returns `201`.
- Amount ≤ 0 or with more than 2 decimal places returns `400`.
- A missing `Idempotency-Key` header returns `400`.
- Repeating the request with the same Idempotency Key credits only once (see FR-8).

### 4.3 Payment (Fast Path)

**Description:** The Payer submits a Transaction to a Payee with their current Coordinates. The Fast Path checks idempotency first, before touching the relational database, then checks the Balance, debits the Payer, records the Transaction as `PRELIMINARY_APPROVED`, guarantees a TransactionCreatedEvent will be emitted, and responds `201`. It does not wait for Risk Analysis. Realizes UJ-1, UJ-2.

#### FR-6: Create a Transaction

An Account Holder can create a Transaction from their own Account to a Payee Account, sending amount, Payee Account id, and Coordinates.

- On success: Payer Balance decreases by the amount, the Transaction is stored as `PRELIMINARY_APPROVED`, and the response is `201` with the Transaction id, status, and a `Location` header.
- Amount ≤ 0 or with more than 2 decimal places returns `400`.
- Payee does not exist returns `404`. Payee equals Payer returns `422`.
- Insufficient Balance returns `422`, and nothing is debited or recorded as a Transaction.
- Missing or out-of-range Coordinates (latitude outside [-90, 90] or longitude outside [-180, 180]) return `400`.
- The Payee is **not** credited at this point. The amount is a Pending Amount until settlement (FR-14).

#### FR-7: Balance stays consistent under concurrency

Concurrent Balance changes on the same Account (Transaction debits, Deposits, Payee credits, Reversals) never drive the Balance negative or lose an update.

- With Balance B and K concurrent requests of amount A where K·A > B, exactly ⌊B/A⌋ succeed and the rest return `422`. Final Balance = B − ⌊B/A⌋·A.
- A Deposit or Reversal running concurrently with debits is never lost: the final Balance equals the sum of all committed changes.
- This is covered by an automated integration test against a real PostgreSQL. The locking strategy is chosen in Architecture (kickoff Q3).

#### FR-8: Idempotent writes

Every Transaction and Deposit request must carry an Idempotency Key. The key, scoped per User, is checked in Redis before the relational database is touched.

- The same key with the same payload within 24h returns the original status code and body, and executes nothing again.
- The same key with a different payload returns `422`.
- The same key while the first request is still in flight returns `409`.
- A missing `Idempotency-Key` header returns `400`.
- If Redis is unavailable, the request is refused with `503` and nothing is written. A write never runs without the idempotency check.
- Only successful (`2xx`) and business-rejection (`4xx`) responses are stored for replay. A `5xx` releases the key so the client can retry.
- Keys expire after 24h.

#### FR-9: Guaranteed event emission

Every Transaction committed as `PRELIMINARY_APPROVED` eventually reaches Risk Analysis, even if the message broker is unavailable when the Transaction is created. It is settled exactly once, or, if analysis keeps failing, parked in the DLQ (FR-18) until replayed.

- With the broker stopped, a Transaction can still be created (`201`). Once the broker is back, its TransactionCreatedEvent is delivered without manual action.
- A Transaction rolled back for any reason emits no event.
- With the Fraud Worker stopped, Transactions are still created (`201`) and stay `PRELIMINARY_APPROVED` until the worker returns; nothing is lost.
- The mechanism (transactional outbox vs. simpler approach) is chosen in Architecture (kickoff Q2).

**Feature-specific NFR:** Fast Path latency target, see NFR-1.

### 4.4 Transaction Status

**Description:** The `201` response comes before Risk Analysis, so the Payer finds out the final outcome by polling. Realizes UJ-1, UJ-2.

#### FR-10: Query a Transaction

An Account Holder can retrieve a Transaction they are Payer of, by id, with its Transaction Status, amount, Payee, and timestamps.

- A Transaction of another Account Holder returns `404`, not `403`, so ids cannot be enumerated.
- The status reflects the Risk Analysis outcome once it is committed.

**Out of Scope:** webhook notifications (§7.2), listing or paging all Transactions `[ASSUMPTION]`.

### 4.5 Risk Analysis (Slow Path)

**Description:** A separate worker consumes each TransactionCreatedEvent, evaluates the three Risk Rules against the Payer History and the Previous Transaction, sums the Risk Score, and settles the Transaction. Below the Fraud Threshold, the Transaction becomes `COMPLETED` and the Payee is credited. At or above it, the Transaction becomes `FRAUD_ALERT` and a Reversal returns the amount to the Payer. Realizes UJ-1, UJ-2.

#### FR-11: Unusual Amount rule

The rule fires when the Transaction amount exceeds the baseline of the Payer History by a configurable factor.

- It is skipped during Cold Start and recorded as `SKIPPED` in the Risk Dossier.
- The baseline definition (for example, a multiple of the mean of the last M Transactions) is configurable. `[ASSUMPTION]`

#### FR-12: Atypical Time rule

The rule fires when the Transaction's time of day falls outside the time-of-day pattern of the Payer History.

- It is skipped during Cold Start.
- The pattern definition (for example, hour-of-day buckets with frequency below X% of history) is configurable. `[ASSUMPTION]`

#### FR-13: Impossible Travel rule

The rule fires when the implied speed between the Coordinates of the Previous Transaction and the current one exceeds a configurable maximum (for example, 900 km/h).

- The rule is skipped when there is no Previous Transaction.
- After UJ-2, Ana's next legitimate Transaction from São Paulo does not fire the rule, because the Berlin Transaction is `FRAUD_ALERT`.
- Two Transactions in Brazil and in Germany a few minutes apart fire the rule.

#### FR-14: Scoring and settlement

The worker sums the points of the fired rules into a Risk Score, compares it with the Fraud Threshold, and transitions the Transaction.

- Risk Score < Fraud Threshold leads to `COMPLETED`, and the Payee Balance increases by the amount.
- Risk Score ≥ Fraud Threshold leads to `FRAUD_ALERT`, and the Reversal increases the Payer Balance by the amount. The Payee is never credited.
- The status transition and the matching Balance change are atomic.

#### FR-15: Exactly-once settlement

Redelivered or retried TransactionCreatedEvents never settle a Transaction twice.

- Delivering the same event twice leaves exactly one status transition, one credit or Reversal, and one Risk Dossier.
- A Transaction that is no longer `PRELIMINARY_APPROVED` is not re-settled.

### 4.6 Risk Dossier (Audit)

**Description:** Every Risk Analysis leaves a record that explains the decision on its own. Realizes UJ-2.

#### FR-16: Write a Risk Dossier

Every settled Transaction has one Risk Dossier containing: Transaction id, Payer, Payee, amount, Coordinates, each Risk Rule's outcome (`FIRED` / `NOT_FIRED` / `SKIPPED`, points, short reason), Risk Score, Fraud Threshold, the Rule Configuration version used, the resulting Transaction Status, and timestamps.

- For any `FRAUD_ALERT`, the Risk Dossier alone shows which Risk Rules caused it.
- Risk Dossiers are append-only. They are never updated after they are written.

**Out of Scope:** an API to query Risk Dossiers `[ASSUMPTION: inspected directly in MongoDB for the MVP]`.

### 4.7 Rule Configuration

**Description:** The Admin tunes detection at runtime. Changes affect Risk Analyses that start after the change. Realizes JTBD Admin.

#### FR-17: View and update Rule Configuration

An Admin can read and replace the Rule Configuration: per-rule weight, Fraud Threshold, Cold Start N, and rule parameters (amount factor, time-pattern parameters, maximum travel speed).

- The update takes effect for new Risk Analyses without a restart or redeploy.
- Each update creates a new Rule Configuration version, which Risk Dossiers reference.
- Invalid values (negative weight, threshold ≤ 0, N < 1) return `400`.
- Account Holders get `403`.
- Sensible defaults exist on first startup.

### 4.8 Resilience

**Description:** A failing Risk Analysis must not lose a Transaction or block the queue. Realizes JTBD Interviewer ("what happens if the worker fails?").

#### FR-18: Retry with exponential backoff, then DLQ

When Risk Analysis of an event fails, it is retried with exponential backoff up to a configurable maximum number of attempts. It then goes to the DLQ.

- A poison message does not block the processing of other events.
- A message in the DLQ is visible in the RabbitMQ management console.
- Replay is manual: the README documents step by step how to move a DLQ message back to the main queue through the RabbitMQ management console. A replayed event settles the Transaction normally, and replaying it twice is safe (FR-15).
- Until replayed, the Transaction stays `PRELIMINARY_APPROVED`: the Payer is debited and the Payee is not credited. This is an accepted MVP limitation, listed in the README.

## 5. Cross-Cutting NFRs

- **NFR-1 — Fast Path latency:** `POST /transactions` p95 < 100 ms on the local `docker compose` stack at a stated load (`[ASSUMPTION]` 50 req/s for 60 s). It is measured by a committed load-test script, and the result is recorded in the repository. The README states the measured number, not just the target. At least one optimization is documented with before/after numbers from the same script.
- **NFR-2 — API contract:** Every endpoint is documented in OpenAPI and browsable in Swagger UI. Every request DTO is validated with `jakarta.validation`.
- **NFR-3 — Error format:** All errors go through one global handler with a single consistent body (status, error code, message, field errors when relevant). No stack traces are exposed.
- **NFR-4 — Security:** JWT signing secrets and admin credentials come from environment configuration. The repository contains only clearly marked dev defaults. Passwords are hashed with an adaptive algorithm.
- **NFR-5 — One-command startup:** `docker compose up` starts PostgreSQL, Redis, MongoDB, RabbitMQ, the Payment API, and the Fraud Worker, with health checks. Seed data is opt-in through a configuration flag and off by default.
- **NFR-6 — Tests:** Unit tests (JUnit 5, Mockito) cover the domain logic, Risk Rules included. Integration tests run against real dependencies through Testcontainers and cover FR-7, FR-8, FR-9, FR-14, FR-15, and FR-18.
- **NFR-7 — CI/CD:** GitHub Actions runs build plus unit and integration tests on every push and pull request. On every version tag, it publishes both service images to GitHub Container Registry. The README shows the status badge.
- **NFR-8 — Observability:** Both services expose health through Actuator. Logs carry the Transaction id across Fast Path and Slow Path, so one Transaction can be followed end-to-end.
- **NFR-9 — Documentation integrity:** The README promises nothing the code does not deliver. Deferred items appear under "Next steps". The README and `CLAUDE.md` describe BMad by its official name, "Breakthrough Method for Agile AI-Driven Development", and by its real workflow (PRD → Architecture → Epics & Stories → Build/Review), not as "Breakdown, Model, Architecture, Deliver".
- **NFR-10 — Delivery process evidence:** Every story is delivered on its own branch through a pull request, merged only with CI green. Commits follow Conventional Commits. The MVP is tagged `v1.0.0`.
- **NFR-11 — AI-assisted process evidence:** The README has a "How AI was used" section. It maps the BMad steps to their artifacts in `_bmad-output/`, points to `CLAUDE.md` as the agent instructions, and describes at least three cases where AI output was reviewed and corrected or rejected.

## 6. Non-Goals (Explicit)

- Not a real payment processor: no card networks, PIX, banks, or settlement with external parties.
- No multi-currency. The MVP uses a single currency (BRL) with 2-decimal amounts. `[ASSUMPTION]`
- No ML or external fraud services. The three rules plus a score are the whole fraud engine.
- No user-initiated refunds, chargebacks, or cancellations.
- No password reset, e-mail verification, MFA, or OAuth2 authorization server.
- No UI.
- No Kafka, service mesh, Kubernetes, or cloud deployment.

## 7. MVP Scope

### 7.1 In Scope

FR-1 through FR-18. NFR-1 through NFR-11. Two deployable services (Payment API, Fraud Worker) plus the infrastructure in one `docker compose`. A Smoke Test script. A load-test script.

### 7.2 Out of Scope for MVP ("Next steps" in the README)

- **Webhook notifications** of the final Transaction Status. Deferred because of their cost: callback registration, retry policy, and HMAC signing. The MVP uses polling (FR-10).
- **Angular UI** for Account Holders and the Admin.
- **DLQ replay tooling** (endpoint or job to reprocess DLQ messages).
- **Both locking strategies.** The MVP implements one and documents why. A benchmark comparison is a possible follow-up.
- **Transaction history listing and Risk Dossier query API.**

### 7.3 Timebox and Cut Order

Personal end date: **2026-12-16** (3 weeks). If the schedule slips, cut in this order:

1. Rule Configuration versioning (FR-17 keeps runtime updates; Risk Dossiers record the values used instead of a version id).
2. Atypical Time rule (FR-12).
3. Load test stays a single measurement; the before/after optimization in NFR-1 is dropped.

## 8. Success Metrics

**Primary**

- **SM-1 — Smoke Test passes on a fresh clone.** `git clone` → `docker compose up` → Smoke Test script exits 0. The script first sets a known Rule Configuration through the Admin endpoint, so the result does not depend on defaults or on the time of day. It covers UJ-1, UJ-2, and an idempotent retry, with no manual steps beyond the documented ones, in under 10 minutes on a typical laptop. Validates FR-1, FR-2, FR-5, FR-6, FR-8, FR-10, FR-13, FR-14, FR-16.
- **SM-2 — Manual QA pass.** Before the project is declared done, the builder runs a written manual QA checklist (every FR's consequences) and records the result. Validates FR-1–FR-18.

**Secondary**

- **SM-3 — CI green** on the default branch, with unit and Testcontainers integration tests. Validates NFR-6, NFR-7.
- **SM-4 — Measured latency.** NFR-1 is met, and the number is published in the README. Validates NFR-1.
- **SM-5 — README ⊆ code.** Every README claim maps to an FR, NFR, or "Next steps" item. Validates NFR-9.

**Counter-metrics (do not optimize)**

- **SM-C1 — Test coverage %.** Chasing a coverage number produces shallow tests. Concurrency, idempotency, and exactly-once tests are worth more. Counterbalances SM-3.
- **SM-C2 — Number of rules or features.** Adding more rules or endpoints dilutes the depth that impresses an interviewer. Counterbalances SM-1.
- **SM-C3 — Latency below target.** Below 100 ms, stop optimizing. Clarity of the Fast Path beats micro-gains. Counterbalances SM-4.

## 9. Open Questions

- **Q-B — Transactional outbox vs. simpler dual-write handling** (kickoff Q2). Owner: Architecture. Must satisfy FR-9.
- **Q-C — Pessimistic vs. optimistic locking** (kickoff Q3). Owner: Architecture. Must satisfy FR-7.
- **Q-D — Default rule numbers:** weights, Fraud Threshold, N, amount factor, time-pattern parameters, maximum travel speed. Owner: builder, during Architecture or the first Risk Analysis story. Non-blocking, because they are configurable (FR-17).

## 10. Assumptions Index

- §3 / FR-3 — The Admin owns no Account and is bootstrapped from environment configuration.
- FR-10 — No listing of all Transactions in the MVP.
- FR-11 / FR-12 — Example baseline and pattern definitions. Final definitions are configurable.
- FR-16 — Risk Dossiers are inspected directly in MongoDB, with no query API.
- NFR-1 — Load profile: 50 req/s for 60 s.
- §6 — Single currency (BRL), 2 decimal places.
