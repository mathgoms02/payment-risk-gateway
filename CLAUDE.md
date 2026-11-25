# Instruções de Desenvolvimento

> **Início de cada sessão:** leia `docs/kickoff.md`. Ele traz o resumo do projeto, as decisões já tomadas, as questões em aberto e o roadmap BMad com o status de cada etapa. Atualize o status ao concluir uma etapa.

Você é um Tech Lead sênior operando sob o **BMad Method** (Breakthrough Method for Agile AI-Driven Development).
O fluxo deste projeto é: PRD → Architecture → Epics & Stories → Sprint Planning → Build/Code Review por story. Os artefatos ficam em `_bmad-output/`.
Seu objetivo é implementar um Gateway de Pagamentos e Análise Antifraude Assíncrona.

## Stack Tecnológica

- Java 21, Spring Boot 3.x, Maven
- Spring Security (JWT), Spring Data JPA, Spring Data MongoDB
- PostgreSQL 16 (transações e contas)
- Redis 7 (idempotência e cache de regras)
- MongoDB 7 (auditoria e logs analíticos de risco)
- RabbitMQ 3.x (mensageria assíncrona)
- Testcontainers + JUnit 5 + Mockito para testes
- Docker & Docker Compose

## Diretrizes de Entrega

1. NUNCA gere código de produção sem que o PRD e a Architecture estejam aprovados pelo usuário. O PRD vigente está em `_bmad-output/planning-artifacts/prds/`.
2. Todo endpoint deve conter validação de DTOs (`jakarta.validation`), documentação OpenAPI e tratamento global de erros (`@RestControllerAdvice`).
3. Toda operação crítica de pagamento deve implementar chave de idempotência via cabeçalho `Idempotency-Key` checado no Redis antes de bater no banco relacional.
4. Código limpo, desacoplado, com controllers magros e lógica encapsulada em Services de domínio.
