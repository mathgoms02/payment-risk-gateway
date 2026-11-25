# Addendum — Payment & Fraud Risk Gateway PRD

Notes too detailed for the PRD that Architecture and the README rewrite still need. Section A feeds Architecture, section B feeds the README and `CLAUDE.md` rewrite, and section C records rejected alternatives.

## A. Inputs for Architecture

- **Q2 (dual write):** FR-9 requires that a committed Transaction is never left without an event, even with the broker down. A transactional outbox (table + relay/poller) satisfies this directly. A "publish after commit" approach does not, unless something reconciles it later. Record the choice as an ADR.
- **Q3 (locking):** FR-7 has a concrete concurrency test. Pessimistic locking (`SELECT … FOR UPDATE`) is simpler to reason about for hot accounts. Optimistic locking (`@Version` + retry) suits low contention. Pick one, write an ADR, and list the other under "Next steps".
- **Settlement write path:** FR-14 requires the worker's outcome (status + credit or Reversal) to land in PostgreSQL atomically. Option 1: the worker writes to PostgreSQL directly (shared database). Option 2: the worker publishes a result event that the Payment API consumes, so each service owns its data. Option 2 is the cleaner microservice design and adds a second queue. Decide in Architecture.
- **Settlement vs. Risk Dossier consistency:** FR-14 writes to PostgreSQL and FR-16 writes to MongoDB. Architecture must say what happens when one write succeeds and the other fails. The FR-15 dedup key must make a retry safe (for example, the dossier is written idempotently by Transaction id).
- **History source for rules:** Unusual Amount, Atypical Time, and Impossible Travel need the Payer's prior Transactions and Coordinates. Decide whether the worker reads them from PostgreSQL, from its own MongoDB projections, or from Redis.
- **Rule Configuration storage:** `CLAUDE.md` names Redis as the rule cache. A plausible design keeps the versioned source of truth in PostgreSQL or MongoDB, caches it in Redis, and invalidates the cache on Admin update.
- **Idempotency in Redis:** `SET key NX` with a 24h TTL for the in-flight marker, then the stored response. A payload fingerprint (hash) detects "same key, different payload".
- **Error body:** RFC 7807 Problem Details (`ProblemDetail` in Spring 6) is a natural fit for NFR-3.
- **Seed:** a Spring profile or a flag (for example, `APP_SEED_ENABLED=false`) controls the seed. Tests enable it.
- **Load test:** k6 script under `/load-test`, with the result summary committed.
- **Code convention:** thin controllers with logic in domain services (`CLAUDE.md` rule 4) carry over as an Architecture convention.

## B. README and `CLAUDE.md` corrections (NFR-9, NFR-11, FR-18, kickoff Q1 and Q5)

**README — fix or remove claims:**

- Replace "BMAD (Breakdown, Model, Architecture, Deliver)" with "BMad Method — Breakthrough Method for Agile AI-Driven Development" (source: GitHub `bmad-code-org/BMAD-METHOD`, About section). Replace the four invented phases with the real workflow used here, and link to `_bmad-output/`.
- "lock pessimista/otimista" → only the strategy chosen in Architecture.
- "Resposta em menos de 100ms" → the measured p95 and how it was measured.
- "Spring Security (OAuth2/JWT)" → "Spring Security (JWT)". There is no OAuth2 authorization server.
- "Java 17+" → Java 21, per the kickoff decision.
- "CQRS-lite" and "validação estática de segurança": remove unless they are built.
- "DLQ para retry automático": the retry is automatic, but the DLQ itself is a parking lot for manual replay.
- "score de reputação, IP": the dossier holds the Risk Score and Coordinates, not IP.
- "CI/CD": accurate once the NFR-7 GHCR publish exists (images pushed to GitHub Container Registry on version tags).
- MIT license: add a `LICENSE` file or drop the claim.

**README — add sections:**

- "Next steps", mirroring PRD §7.2.
- "How AI was used" (NFR-11).
- Manual DLQ replay procedure, step by step through the RabbitMQ management console (FR-18).

**`CLAUDE.md`:**

- Fix the BMad name and phases, as in the README.
- Java 17+ → 21.
- Replace the "Breakdown/Model" gate in rule 1 with "PRD/Architecture approved".

## C. Rejected alternatives

- **Webhook in the MVP:** rejected for its cost (callback URL registration, retry/backoff, HMAC signing). Polling covers the need.
- **Fixed-threshold rules:** rejected in favor of per-Payer history with Cold Start. More realistic, and easier to defend in an interview.
- **Any single rule ⇒ fraud:** rejected in favor of a weighted Risk Score.
- **Rules editable only via Redis or config files:** rejected in favor of an Admin endpoint, which demonstrates role-based authorization.

---

Finalize review sources for the items above: `reconcile-*.md`, `review-rubric.md`, `review-adversarial.md` in this folder.
