# Project Kickoff — Payment & Fraud Risk Gateway

> Starting context for every new AI session. Read this before running any BMad skill.
> Last updated: 2026-11-25 (after PRD)

## 1. Goal

Portfolio project for a **Junior Backend Software Engineer** position (Java 17+, Spring Boot/Data/Security,
PostgreSQL, MongoDB, Redis, Kafka/RabbitMQ, microservices, CI/CD, AI-assisted development).

The aim is to impress technical interviewers. The repository must show **working code** and the
**documented AI-assisted process** (BMad Method artifacts under `_bmad-output/`).

## 2. Product summary

An anti-fraud payment gateway with asynchronous risk analysis.

- **Payment API** (Spring Boot 3, Java 21)
  - `POST /transactions`, authenticated with JWT
  - Checks the `Idempotency-Key` header in Redis before touching PostgreSQL
  - Validates the account balance in PostgreSQL
  - Saves the transaction as `PRELIMINARY_APPROVED`
  - Publishes `TransactionCreatedEvent` to RabbitMQ
  - Responds `201 Created`
- **Fraud Worker** (Spring Boot)
  - Consumes `TransactionCreatedEvent`
  - Applies risk rules: unusual amount, atypical time of day, mismatched geolocation
  - Writes the audit dossier to MongoDB with the result `COMPLETED` or `FRAUD_ALERT`
- **Infrastructure:** everything starts with `docker compose up`
- **Quality:** JUnit 5, Mockito, Testcontainers; API documented with OpenAPI

## 3. Constraints

- Solo developer, junior level. A lean, well-built MVP, not a big system.
- No hard deadline. The first estimate was 2 weeks; about 3 weeks is acceptable on the full track.
  Set a personal end date anyway so the scope stays under control.
- Communication with the AI happens in **Portuguese**. Code, technical names and BMad documents are in **English**.

## 4. Decisions made

| Decision | Choice | Reason |
|---|---|---|
| BMad track | Full BMad Method (lean version) | No hard deadline. PRD → architecture → stories gives traceability to show in the interview |
| Skipped steps | Product Brief, UX, PRFAQ, Brainstorming | README already works as the brief, there is no UI, and the concept is already decided |
| Java version | 21 | LTS. CLAUDE.md says 17+, so 21 still complies |
| Messaging | RabbitMQ only | Kafka would add scope without adding learning value for this MVP |
| Domain naming | English (`PRELIMINARY_APPROVED`, `FRAUD_ALERT`, `COMPLETED`, `/transactions`) | Code is in English |

## 5. Open questions (to settle during the PRD / Architecture)

> **Status after PRD (2026-11-25):** Q1, Q4 and Q5 are settled in the PRD (§7, FR-2, NFR-9). Q2 and Q3 became requirements (FR-9, FR-7); the mechanism choice is still open for Architecture. The personal end date is 2026-12-16 (PRD §7.3).

1. **README scope vs. MVP.** The README promises a DLQ with exponential backoff, OAuth2/JWT,
   pessimistic *and* optimistic locking, latency under 100ms and GitHub Actions. Decide what is in the MVP
   and move the rest to a "Next steps" section. The README must not promise more than the code delivers.
2. **Dual write (PostgreSQL + RabbitMQ).** Choose between the transactional outbox and a simpler approach,
   and record the reasoning (Architecture step).
3. **Locking strategy** for balance updates: pessimistic or optimistic (Architecture step).
4. **Where JWTs come from.** Options: a simple login endpoint, or a fixed dev key (PRD/Architecture).
5. **Naming of BMAD.** CLAUDE.md and the README explain BMAD as "Breakdown, Model, Architecture, Deliver".
   The official name is believed to be *Breakthrough Method for Agile AI-Driven Development*. Check it at
   https://docs.bmad-method.org/ and fix it.

## 6. BMad roadmap

Run each step in a **fresh session** (`/clear` and then the skill command).

| # | Step | Skill | Required | Status |
|---|---|---|---|---|
| 1 | PRD | `bmad-prd` | yes | ✅ done 2026-11-25 — `_bmad-output/planning-artifacts/prds/prd-payment-risk-gateway-2026-11-25/prd.md` |
| 2 | Architecture | `bmad-architecture` | yes | ⬜ |
| 3 | Test Design | `bmad-testarch-test-design` | optional (recommended) | ⬜ |
| 4 | Epics and Stories | `bmad-create-epics-and-stories` | yes | ⬜ |
| 5 | Sprint Planning | `bmad-sprint-planning` | yes | ⬜ |
| 6 | Build → Code Review (one cycle per story) | `bmad-build` → `bmad-code-review` | yes | ⬜ |
| 7 | Retrospective (at the end of each epic) | `bmad-retrospective` | optional | ⬜ |

**Next action:** run `/bmad-architecture` with the PRD and its `addendum.md` (§A lists the Architecture inputs: Q2 outbox, Q3 locking, settlement write path).
