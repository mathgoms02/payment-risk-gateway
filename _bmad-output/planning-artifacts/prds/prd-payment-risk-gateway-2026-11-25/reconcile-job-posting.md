---
title: "Input reconciliation — Job posting vs. PRD"
input: "Junior Backend Software Engineer job posting (pt-BR, verbatim in the request)"
prd: prd.md (draft, 2026-11-25)
created: 2026-11-25
---

# Input reconciliation: job posting vs. PRD

**Lens:** the project exists to give a junior candidate *concrete, demonstrable* evidence for each posting item, evidence an interviewer can see in the repo, the README, or a 10-minute demo. "Mentioned in the stack" does not count. "Exercised by an FR/NFR and visible in an artifact" does.

**Constraints respected (from .memlog.md / kickoff):** RabbitMQ only, no Kafka. No MySQL. Solo junior dev, lean MVP (~3 weeks). SM-C2 (do not add features) holds. Every recommendation below is a PRD-level requirement or acceptance criterion that costs hours, not days, and adds no new runtime feature.

## 1. Item-by-item verdict

Legend: **Strong** = the PRD forces demonstrable evidence. **Partial** = present but implicit, unverifiable, or easy to lose in Architecture. **Gap** = the PRD gives no requirement that would produce evidence.

| # | Posting item | Verdict | PRD evidence | What's missing |
|---|---|---|---|---|
| 1 | Desenvolvimento de APIs RESTful | **Strong** | FR-1..FR-17 with explicit status codes (201/400/401/403/404/409/422), `Location` header (FR-6), 404-not-403 anti-enumeration (FR-10), NFR-2 OpenAPI/Swagger, NFR-3 single error format | Nothing material. |
| 2 | **Otimização** de APIs e serviços | **Partial (weak)** | NFR-1 p95 < 100 ms measured by a committed k6 script, number in the README (SM-4) | The PRD proves a target was *met*. It never proves something was *optimized*. SM-C3 explicitly says "stop optimizing", so the likely outcome is one number with no before/after. No requirement covers indexes, query plans, Redis-as-cache impact, or connection pool tuning. See G1. |
| 3 | Apoio de IA Generativa na codificação | **Partial** | §2.1 Builder JTBD ("AI-assisted process (BMad artifacts)"), NFR-9 (BMad named correctly, real workflow described) | NFR-9 only fixes the *name*. Nothing requires visible evidence of *how* AI was used: what the agent did, what the human reviewed, rejected, or rewrote. An interviewer will ask "what did *you* do vs. the AI?" and the PRD produces no artifact that answers it. See G2. |
| 4 | Java 17+ | **Strong** | Stack (Java 21 per addendum B) | None. Optional: one line in the README naming the modern-Java features actually used (records for DTOs/events, sealed types or switch patterns for rule outcomes). Architecture-level, not PRD. |
| 5 | Spring Boot | **Strong** | Two Boot services (NFR-5), Actuator (NFR-8) | None. |
| 6 | Spring Data | **Partial** | Implied: PostgreSQL via JPA (accounts, transactions), MongoDB (dossiers) | The PRD never says Spring Data JPA / Spring Data MongoDB. Addendum A leaves the dossier store and rule-config store open. Low risk, since CLAUDE.md fixes the stack, but the README should name both modules (see §2 cheap fixes). |
| 7 | Spring Security | **Strong** | FR-2 JWT, 401 vs 403 distinction, FR-3 Admin role, FR-17 role-based 403, NFR-4 hashed passwords and env secrets | None. |
| 8 | SQL (PostgreSQL, MySQL) | **Strong for PostgreSQL**; MySQL deliberately absent | FR-7 concurrency test against real PostgreSQL, FR-14 atomic status+balance, locking ADR (Q-C) | Absence of MySQL is a correct decision. Cheap talking point: one README line stating the schema avoids PostgreSQL-only types so it is portable to MySQL. Do not add MySQL. Schema migrations are not required anywhere (see G5). |
| 9 | NoSQL: MongoDB | **Partial** | FR-16 append-only Risk Dossier with flexible per-rule outcomes | Dossiers have no read API (deferred, correct). But SM-1's Smoke Test is not required to *show* a dossier, so the interviewer never sees MongoDB doing anything unless they open `mongosh`. See G4. |
| 10 | NoSQL: Redis | **Partial** | FR-8 idempotency checked before relational write (24h TTL) | The Redis *rule cache* exists only in CLAUDE.md and addendum A ("a plausible design"). No FR requires it. If Architecture drops it, Redis shows only one use. The cache is also the natural vehicle for the "otimização" evidence (G1). |
| 11 | APIs RESTful **e padrões de integração** | **Strong (content), Partial (visibility)** | FR-9 guaranteed emission (transactional outbox candidate), FR-15 idempotent consumer, FR-18 retry + backoff + DLQ, FR-8 idempotency key, Q-B/Q-C as ADRs | The patterns are there but the PRD never requires them to be *named*. An interviewer scanning the README should see "Transactional Outbox, Idempotent Consumer, Dead Letter Channel, Retry with Exponential Backoff, Idempotency Key" with a pointer to the ADR/test for each. Cheap fix, see §2. |
| 12 | Mensageria (Kafka, RabbitMQ) | **Strong for RabbitMQ**; Kafka deliberately absent | FR-9, FR-15, FR-18, DLQ visible in the management console, broker-down test | Kafka absence is correct (kickoff decision). Cheap talking point: a short "Why RabbitMQ, not Kafka" note in the README or an ADR (work-queue semantics, per-message ack, native DLX vs. log replay/partitions). This turns an absence into evidence of judgment. |
| 13 | Arquitetura de microsserviços | **Partial (at risk)** | Two deployable services (Payment API, Fraud Worker), fast/slow path separation (§1) | Addendum A leaves open whether the worker writes the Payment API's PostgreSQL directly (shared DB). A shared-database "microservice" is a known anti-pattern, and interviewers probe it. The PRD does not need to choose, but it should require that the choice and its trade-off are recorded in an ADR and surfaced in the README. See G5. |
| 14 | Git | **Gap** | Nothing | No requirement on branching, PRs, commit conventions, or tags. The repo currently has **zero commits**. For a junior, the commit history *is* the Git evidence. See G3. |
| 15 | Metodologias ágeis (Scrum/Kanban) | **Partial** | BMad (an agile AI-driven method) produces Epics & Stories and sprint planning (kickoff roadmap step 5) | The stories will live as files in `_bmad-output/`. Nothing requires them to be visible as a board, or traceable from story → branch/PR → commit, or a retrospective. See G3. |
| 16 | CI/CD | **Partial (CI only)** | NFR-7 GitHub Actions build + unit + Testcontainers tests on push/PR, badge, SM-3 | Only **CI**. No **CD** at all: §6 rules out cloud deployment (correct), but delivery does not require cloud. Publishing the two service images to GitHub Container Registry on merge to the default branch (plus a version tag) is cheap and makes "CD" literally true. See G6 in §3. |
| 17 | Ferramentas de IA: assistentes de código, engenharia de prompts, **agentes** | **Partial** | BMad artifacts in `_bmad-output/`, CLAUDE.md in the repo | Same root cause as item 3. The posting names three things (assistants, prompt engineering, agents). The repo already *has* them (CLAUDE.md = project-level prompt engineering, BMad skills = agents, `.claude/`), but no requirement makes them legible to someone who has never heard of BMad. See G2. |

## 2. Cheap fixes that need no new requirement (README / Architecture notes)

These belong in the README rewrite (addendum B) or Architecture. They do not need FRs.

- Name the Spring Data modules used (JPA, MongoDB) and the Java 21 features actually used.
- A "Integration patterns" table: pattern → where (FR) → proof (test class or ADR).
- "Why RabbitMQ, not Kafka" and "Why PostgreSQL; MySQL-portable schema" as two short paragraphs or ADRs. They answer the two posting items the project deliberately skips.

## 3. Gaps where a cheap PRD-level requirement materially strengthens evidence

Ordered by interview impact. Each proposal is phrased as a requirement the PRD could adopt as-is.

### G1. "Otimização" has no evidence, only a target (high impact)

*Problem.* The posting's first sentence says "desenvolvimento **e otimização**". NFR-1 yields one number, and SM-C3 tells the builder to stop once under 100 ms. An interviewer asking "tell me about something you optimized" gets nothing.

*Proposed NFR (amend NFR-1 or add NFR-10).* "The load-test result in the repository includes at least one before/after comparison for a named optimization (for example: a missing index on the Payer history query, the Redis rule-configuration cache vs. a database read per analysis, or connection-pool sizing). The README states the change, both numbers, and the method." Keep SM-C3: one documented optimization, then stop.

*Cost.* One extra k6 run and a paragraph. It reuses NFR-1's script.

### G2. AI-assisted process is claimed, not shown (high impact)

*Problem.* The posting asks for GenAI-assisted coding, prompt engineering, and agents. NFR-9 only corrects the BMad name. Without evidence of human judgment over AI output, "built with AI" reads as a liability to interviewers rather than a skill.

*Proposed NFR-10/11 (documentation).* "The README has an 'AI-assisted development' section that (a) maps the BMad steps actually run to their artifacts in `_bmad-output/`, (b) points to the project's agent instructions (`CLAUDE.md`) as the prompt-engineering artifact, and (c) lists at least three concrete cases where AI output was reviewed and corrected or rejected by the builder, with links to the commit, PR, or review finding." Optionally: each story's PR description states what the agent produced and what the builder changed.

*Cost.* Low. The artifacts already exist. This is curation, done as the work happens. Note: the `.memlog.md` decision trail is itself good evidence (for example, the BMad-name correction and the webhook reversal) and could be linked.

### G3. Git and agile practice are invisible (high impact)

*Problem.* No requirement covers Git workflow, and the repo has no commits yet. BMad stories will exist only as files. "Scrum/Kanban" and "Git" are two of seven posting lines, and the PRD produces no artifact for either.

*Proposed NFR (process).* "Work is tracked as one GitHub Issue per BMad story on a GitHub Projects Kanban board. Each story is delivered through a branch and a pull request that references its issue and passes CI before merge. Commits follow Conventional Commits. The MVP is tagged `v1.0.0` with a release note." Optional: one short retrospective file at the end, which BMad already supports (`bmad-retrospective`).

*Cost.* Minutes per story. It also makes NFR-7 (CI on PR) meaningful, since today there would be no PRs to run on. Branch-protection on the default branch is optional and free.

### G4. MongoDB and Redis evidence are not demonstrable in the demo (medium impact)

*Problem.* SM-1's Smoke Test covers UJ-1, UJ-2, and an idempotent retry, but it is not required to show the Risk Dossier. The interviewer never sees MongoDB at work. The Redis rule cache is not an FR at all, so Redis may end up with a single use.

*Proposed acceptance criteria (no new endpoint).*
- SM-1 / UJ-3: "For the UJ-2 Transaction, the Smoke Test prints the Risk Dossier from MongoDB (for example via `mongosh` in the container) showing which rules fired."
- FR-17: "Rule Configuration reads on the Slow Path are served from a Redis cache that is invalidated on Admin update." This lifts the existing CLAUDE.md/addendum intent into a testable requirement. It also gives G1 its natural before/after.

*Cost.* A few lines in the smoke script. The cache was already planned.

### G5. Microservice boundary and schema evolution are unspecified (medium impact)

*Problem.* Addendum A leaves "worker writes PostgreSQL directly (shared DB)" open. A shared database between two "microservices" is the first thing an interviewer will challenge. Separately, no requirement covers database migrations, so the schema could end up as `ddl-auto=update`, which reads as junior in a code review.

*Proposed requirements.*
- Addendum/Architecture input made mandatory: "The settlement write-path decision (shared DB vs. result event) is recorded in an ADR with its trade-off, and the README states which one was chosen and why." The PRD should not force the result-event option. A well-argued shared DB is acceptable for an MVP if it is stated honestly.
- NFR: "The relational schema is managed by versioned migrations (Flyway or Liquibase). Hibernate auto-DDL is disabled outside tests."

*Cost.* One ADR, plus Flyway, which takes about an hour to set up.

### G6. CI/CD is only CI (lower impact, very cheap)

*Proposed amendment to NFR-7.* "On merge to the default branch, and on version tags, the pipeline builds and publishes both service images to GitHub Container Registry. `docker compose` can run from published images." This stays within §6: no cloud deployment.

*Cost.* One workflow job.

## 4. Deliberately absent items (no action, confirm only)

- **Kafka:** correctly excluded (kickoff, §6). Convert to evidence with the "Why RabbitMQ" note (§2).
- **MySQL:** correctly excluded. One portability line in the README (§2).
- **Production cloud deployment, Kubernetes:** correctly excluded (§6). G6 delivers "CD" without them.

## 5. Scope-creep guardrails (rejected ideas)

Considered and rejected as not worth the cost for a 3-week solo MVP:
- Adding a Kafka variant, a MySQL profile, or a dossier query API "for the posting". These are explicit non-goals and would breach SM-C2.
- Distributed tracing (Zipkin/Tempo). NFR-8's correlated Transaction-id logs are enough evidence.
- A third microservice (for example, a notification service). Webhooks stay in "Next steps".
- Coverage gates. SM-C1 already rejects them.

## 6. Summary

The PRD is strong on the technical core of the posting: REST, Spring Security, PostgreSQL concurrency, RabbitMQ integration patterns, and tests/CI. Its weak spots are the *process* items, which are cheapest to fix and most visible to an interviewer: optimization evidence, visible AI-assisted workflow, and Git/agile traceability. Adopting G1–G3 as NFRs and G4 as acceptance criteria adds roughly one to two days across the MVP and no runtime scope.
