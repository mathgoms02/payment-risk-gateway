# Input Reconciliation — docs/kickoff.md vs PRD + addendum

- **Input:** `docs/kickoff.md` (last updated 2026-11-25)
- **Compared against:** `prd.md`, `addendum.md` (same folder), with `.memlog.md` used to separate deliberate changes from silent drops
- **Date:** 2026-11-25

## Summary

The PRD covers the kickoff well. All five §5 open questions are either resolved (Q1, Q4, Q5) or turned into capability requirements and handed to Architecture (Q2 → FR-9, Q3 → FR-7). Every item in the §2 product summary maps to an FR or NFR. The deliberate changes are logged in the memlog and are not flagged here: debit at the Fast Path with the hold model, the worker settling in PostgreSQL with a Reversal, webhooks deferred, impossible travel as the geolocation rule, and the Admin role. The gaps below are mostly about **qualitative intent** (lean scope, time budget, the "documented AI process" half of the goal) and a few precise wordings that got weaker on the way.

## Gaps (ordered by importance)

### G1 — The time budget was dropped, and the scope grew with no check against it (qualitative intent, §3)

The kickoff says: "Solo developer, junior level. A lean, well-built MVP, not a big system." It gives a first estimate of 2 weeks, with about 3 weeks acceptable.

The PRD never mentions the 2–3 week budget. Q-E (personal end date) is still open with no target. Meanwhile, the MVP grew well past the kickoff's two-service sketch: registration, login, an Admin role and its bootstrap, deposits, versioned Rule Configuration, per-Payer history rules with Cold Start, a score and threshold, settlement and Reversal, exactly-once settlement, an outbox-grade delivery guarantee, retry/DLQ, CI, a load-test script, a Smoke Test script, and a manual QA checklist (18 FRs + 9 NFRs). Each addition is a logged decision. Nothing in the PRD tests their sum against the budget, though, and there is no priority order (must-have vs. cut-first) to fall back on if the budget runs out. SM-C2 guards against adding *more* rules or endpoints, but not against the scope already in place.

**Suggestion:** state the time budget in §7 or §9, resolve Q-E with a date, and tag FRs/NFRs with a cut order (for example, NFR-1 load test, FR-17 versioning, and FR-12 Atypical Time as first candidates to cut).

### G2 — The "documented AI-assisted process" half of the goal has no requirement or metric (§1)

The kickoff says: "The repository must show **working code** and the **documented AI-assisted process** (BMad Method artifacts under `_bmad-output/`)."

The PRD covers the working-code half thoroughly (SM-1 to SM-4). The AI-process half appears only in the Builder JTBD and in NFR-9's wording fix for the BMad name. No requirement says that `_bmad-output/` artifacts are committed and kept current, or that the README links to them and explains the flow. The link appears only in addendum §B. No success metric checks it either. `_bmad-output/` is currently untracked in git.

**Suggestion:** add to NFR-9 (or a new NFR) "the BMad artifacts (PRD, architecture, ADRs, epics/stories, reviews) are committed and linked from the README", and add a matching success metric (for example, SM-6: an interviewer can trace one FR → story → commit).

### G3 — The missing `Idempotency-Key` on `POST /transactions` is not specified, and "before touching PostgreSQL" became "before any relational write"

The kickoff says: "Checks the `Idempotency-Key` header in Redis **before touching PostgreSQL**." `CLAUDE.md` rule 3 applies the same rule to every critical payment operation.

- FR-5 (Deposit) says a missing header returns `400`. FR-6 and FR-8 say nothing about a missing header on a Transaction, which is the operation the kickoff cares about most. Read literally, the header could be optional for payments.
- FR-8 says "before any relational **write**". The §4.3 description says "before touching the relational database". The kickoff (and CLAUDE.md) mean the stricter version: no PostgreSQL access at all, including the balance read. The two PRD statements disagree with each other, and the FR is the weaker one.
- Redis as the idempotency store appears only in the addendum. That is acceptable as a mechanism, but CLAUDE.md treats it as a hard constraint, so it can be stated as one.

**Suggestion:** in FR-6/FR-8, add "missing `Idempotency-Key` returns `400`", and change FR-8 to "before any relational read or write".

### G4 — BMad naming (Q5) was verified against a different source than the one the kickoff named, and confirmed only implicitly

The kickoff says to check the name at https://docs.bmad-method.org/ and fix it. The addendum cites the GitHub `bmad-code-org/BMAD-METHOD` About section instead. The memlog records the confirmation as "(assumption) … confirmed by user implicitly (no objection)". NFR-9 now makes the name a hard requirement. The content is probably right, but the verification step the kickoff asked for was not done as specified, and the user never explicitly approved it.

**Suggestion:** check docs.bmad-method.org, then record an explicit confirmation in the memlog and in the addendum §B source.

### G5 — Minor: kickoff decisions that live only in the addendum, or nowhere

- **Java 21** (kickoff §4 decision, deliberately chosen over CLAUDE.md's "17+"): it appears only as an addendum README correction. The PRD has no tech-constraints line, so Architecture could miss it. NFR-9 also asks for CLAUDE.md fixes, but only the BMad name is listed. The "Java 17+" line in CLAUDE.md is not.
- **RabbitMQ as the broker:** the PRD names RabbitMQ only indirectly (FR-18 "RabbitMQ management console", §6 "No Kafka"). It is fine, but implicit.
- **MongoDB as the dossier store** (kickoff §2): it appears only in the FR-16 assumption tag. It is not stated as a constraint.
- **CI/CD:** the job posting and kickoff §1 say "CI/CD". NFR-7 covers CI only, and nothing about CD is listed as in scope or out of scope. §6 excludes cloud deployment, but something like publishing images to a registry is neither in nor out. Add it to §7.2 or state it as a non-goal, so the README does not over-claim "CI/CD".
- **Roadmap housekeeping:** CLAUDE.md says to update the kickoff status when a step finishes. Kickoff §6 still shows step 1 (PRD) as "⬜ not started". Update it when the PRD is finalized. This does not affect the PRD content.

## Verified as covered or deliberately changed (no action)

| Kickoff item | Where it landed |
|---|---|
| `POST /transactions`, JWT | FR-6, FR-2, NFR-1 |
| Balance validation in PostgreSQL | FR-6, FR-7 (extended to debit + hold model, memlog) |
| Status `PRELIMINARY_APPROVED`, `201` | FR-6 |
| Publish `TransactionCreatedEvent` | FR-9 (strengthened to guaranteed delivery) |
| Worker consumes the event and applies 3 rules | FR-11–FR-14 (geolocation → Impossible Travel, memlog) |
| Dossier with `COMPLETED`/`FRAUD_ALERT` | FR-16; worker also settles in PostgreSQL (memlog, deliberate) |
| `docker compose up` | NFR-5, SM-1 |
| JUnit 5, Mockito, Testcontainers, OpenAPI | NFR-6, NFR-2 |
| English naming | Glossary, statuses, paths |
| RabbitMQ only, no Kafka | §6 non-goals |
| Q1 README scope (DLQ, OAuth2, locks, <100 ms, GH Actions) | FR-18, addendum §B, §7.2, NFR-1, NFR-7, NFR-9 |
| Q2 / Q3 | FR-9 / FR-7 + addendum §A, open as Q-B / Q-C (deliberate) |
| Q4 JWT source | FR-2 login endpoint (memlog) |
| Q5 BMad name | NFR-9, addendum §B (see G4) |
| Personal end date | Q-E (open; see G1) |
| Webhook | Deferred, §7.2 (memlog, deliberate) |

## Incidental (not a kickoff gap, noticed during the comparison)

FR-9 promises that every committed Transaction "eventually produces exactly one Risk Analysis outcome". FR-18, however, leaves DLQ'd Transactions in `PRELIMINARY_APPROVED`, with their Pending Amount held, until someone replays them manually. Reword FR-9 to except DLQ'd events, or state that the held funds stay held until replay.
