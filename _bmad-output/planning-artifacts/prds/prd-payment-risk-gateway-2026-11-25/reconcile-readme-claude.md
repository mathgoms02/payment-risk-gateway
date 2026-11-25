---
title: "Input reconciliation — README.md + CLAUDE.md vs PRD"
created: 2026-11-25
inputs: [README.md, CLAUDE.md]
compared_against: [prd.md, addendum.md, .memlog.md]
---

# Input reconciliation: README.md and CLAUDE.md

Method: every README promise and CLAUDE.md rule was checked against (a) PRD FR/NFR, (b) PRD §7.2 "Next steps" / non-goals, (c) addendum §B corrections, (d) `.memlog.md` decisions. Only items that fall through all four are listed as gaps. Covered items are listed at the end for traceability.

## Gaps (ordered by importance)

### G1. "CQRS-lite" and "validação estática de segurança" are claims with no home  — HIGH
README BMAD section, step 3 claims "segregação de responsabilidades (CQRS-lite)"; step 4 claims "validação estática de segurança e automação de pipelines".
- No FR/NFR describes a separate read model (FR-10 reads the Transaction status; nothing segregates reads from writes). No SAST / dependency scan (CodeQL, OWASP Dependency-Check, SpotBugs) in NFR-7, which is build + tests only.
- Addendum §B says "replace the 4 invented phases with the real flow", but does not say what happens to the technical claims embedded in those phases. A rewrite that only renames phases could keep them. SM-5 / NFR-9 ("README ⊆ code") would then fail.
- Action: either add an explicit §B line "drop CQRS-lite and static security validation" (or move them to Next steps), or add an NFR (e.g. CodeQL/dependency scan in CI) if wanted for the job posting's CI/CD keyword.

### G2. "CI/CD" promised, PRD delivers only CI — HIGH
README stack lists "GitHub Actions (CI/CD)" and step 4 "automação de pipelines". NFR-7 covers CI only; no deployment/delivery stage (image build/push, release) exists, and §6 excludes cloud deployment. Not corrected in §B. Either rewrite to "CI" in §B, or define a minimal CD (e.g. build and publish Docker images to GHCR on main) as an NFR or Next step.

### G3. Risk dossier content: "score de reputação" and "IP" — MEDIUM-HIGH
README: MongoDB dossiers store "score de reputação, IP, geolocalização e heurísticas avaliadas". FR-16 lists Coordinates, rule outcomes, Risk Score — no reputation score and no client IP (FR-6 request does not capture IP). Not in §B, not deferred. Either add IP capture to FR-6/FR-16 (cheap, useful for audit) and define "reputation score" (or rename it to Risk Score), or add a §B correction.

### G4. README "DLQ para retry automático" contradicts FR-18 — MEDIUM
README: "fila principal e Dead Letter Queue (DLQ) para retry automático com backoff exponencial". FR-18 retries *before* the DLQ and makes DLQ replay manual `[ASSUMPTION]`; §7.2 defers DLQ replay tooling. The README sentence reads as automatic retry *from* the DLQ. Not corrected in §B. Add a §B line rewording it (retry with backoff, then park in DLQ; manual replay).

### G5. "Tolerante a falhas" — behaviour when Redis / MongoDB / PostgreSQL is down is unspecified — MEDIUM
README opens with "sistema distribuído e tolerante a falhas". The PRD covers broker outage (FR-9) and worker failure (FR-18) only.
- Redis down: CLAUDE.md rule 3 makes Redis the idempotency gate on every critical write; no FR says whether the Fast Path fails closed (503) or proceeds without idempotency. Also affects Rule Configuration cache reads in the worker.
- MongoDB down: is settlement (FR-14, PostgreSQL) blocked until the dossier (FR-16) is written, or can they diverge? FR-15 "exactly one Risk Dossier" implies a coupling not stated.
- Action: add acceptance criteria (e.g. FR-8: Redis unavailable ⇒ `503`, nothing written) or soften the README claim in §B.

### G6. CLAUDE.md rule 3 is weakened in FR-8 — MEDIUM
CLAUDE.md: Idempotency-Key "checado no Redis antes de bater no banco relacional" (before *any* relational access). FR-8 says "before any relational write"; only the §4.3 description says "before touching the relational database". FR-8 also does not name Redis (only addendum §A does). Align FR-8 wording with §4.3 / CLAUDE.md, or record a deliberate relaxation in `.memlog.md`.

### G7. Java version and stale BMAD gate in CLAUDE.md — MEDIUM-LOW
- §B fixes "Java 17+ → 21" for the README only; CLAUDE.md "Stack" still says Java 17+. Kickoff says 21 complies, but the two docs will disagree after the README rewrite.
- CLAUDE.md rule 1 ("never generate production code before validating Breakdown and Model") depends on the invented phase names. §B renames BMAD in CLAUDE.md but does not say how this gate is restated (e.g. "no production code before PRD + Architecture are approved"). If only the name changes, the rule references phases that no longer exist.

### G8. MIT license — LOW-MEDIUM
README: "Distribuído sob a licença MIT." No LICENSE file exists in the repo, no NFR/§B item covers it. Under SM-5 every README claim must map to something. Add to NFR-9 / §B ("commit LICENSE") or drop the line.

### G9. "Thin controllers, domain services" (CLAUDE.md rule 4) — LOW
Code-structure rule with no PRD/addendum trace. Acceptable to leave out of the PRD, but it should be forwarded explicitly to Architecture / project-context so it is not lost. Suggest an addendum §A bullet.

### G10. Operational URLs, credentials and versions — LOW
- Swagger at `http://localhost:8080/swagger-ui.html`, Actuator at `:8080/actuator/health`: NFR-2/NFR-8 require Swagger UI and Actuator but fix no port/path. SM-1 depends on documented steps, so the README URLs become a de facto contract; the worker's health port is not mentioned at all.
- RabbitMQ console `guest/guest`: FR-18 relies on the console; NFR-4 allows "clearly marked dev defaults" but does not list broker credentials. Make them env-configurable and mark guest/guest as dev-only.
- Versions PostgreSQL 16, Redis 7, MongoDB 7, RabbitMQ 3.x, Maven 3.9+ and `./mvnw clean verify`: no NFR pins them (NFR-5 names components only; NFR-6/7 do not require the Maven wrapper). Low risk; pin in Architecture.
- "Java 17+ e Maven opcional para rodar sem container": running without containers is not a requirement anywhere; harmless but unverified.

### G11. "Logs analíticos de risco" in MongoDB (CLAUDE.md) — LOW
CLAUDE.md assigns MongoDB to "auditoria e logs analíticos de risco". FR-16 covers the dossier only; no analytics/log collection and no dossier query API (§7.2 defers it). Either treat the dossier as the whole meaning or note the narrowing in §B / addendum.

## Covered (no action)

| Input claim / rule | Where covered |
|---|---|
| Fast path < 100 ms | NFR-1, §B (measured p95) |
| Valida saldo | FR-6 |
| Lock pessimista/otimista | FR-7, Q-C, §B, §7.2 |
| Idempotência Redis TTL 24h | FR-8, addendum §A |
| RabbitMQ fila + DLQ + backoff exponencial | FR-18 (see G4 for wording) |
| Fraud worker desacoplado / EDA / microsserviços | Vision, FR-9, §4.5, §7.1 (two services, TransactionCreatedEvent) |
| Auditoria MongoDB | FR-16 (see G3, G11) |
| Spring Security OAuth2/JWT | FR-2, §6, §B (drop OAuth2) |
| JPA, Postgres, Mongo, Redis, RabbitMQ | NFR-5, addendum §A |
| JUnit 5, Mockito, Testcontainers | NFR-6 |
| Docker Compose one-command | NFR-5, SM-1 |
| GitHub Actions | NFR-7 (see G2) |
| BMAD naming / Claude Code | NFR-9, §B, memlog Q5 |
| DTO validation (`jakarta.validation`) | NFR-2 |
| OpenAPI documentation | NFR-2 |
| `@RestControllerAdvice` global errors | NFR-3, addendum §A (ProblemDetail) |
| Redis rule cache | FR-17 + addendum §A, memlog |
| Integration tests without mocks | NFR-6 |
| Actuator health | NFR-8 (see G10) |
| Swagger UI | NFR-2 (see G10) |
