# Payment & Fraud Risk Gateway

Backend de pagamentos com análise antifraude assíncrona. A API responde ao comprador em milissegundos, sem esperar pela análise de risco. Um worker separado roda essa análise logo em seguida e, se detectar fraude, marca a transação e devolve o dinheiro ao pagador.

> **Status:** 🚧 em desenvolvimento. O planejamento (PRD) está concluído e a próxima etapa é a Architecture. O código ainda não foi escrito. Este README descreve o **escopo planejado** do MVP e só vai afirmar como pronto o que o código realmente entregar.

---

## Como funciona

**Fast path (Payment API, síncrono)**
1. `POST /transactions` autenticado com JWT.
2. O cabeçalho `Idempotency-Key` é checado no **Redis** antes de tocar no PostgreSQL.
3. O saldo é validado e o pagador é debitado no **PostgreSQL**, com o saldo correto mesmo sob requisições concorrentes.
4. A transação é gravada como `PRELIMINARY_APPROVED`, e o evento `TransactionCreatedEvent` é garantido no **RabbitMQ**, mesmo se o broker estiver fora no momento.
5. A resposta é `201 Created`.

**Slow path (Fraud Worker, assíncrono)**
1. O worker consome o evento e avalia três regras, cada uma somando pontos num **risk score**:
   - **Valor incomum** em relação ao histórico do próprio pagador.
   - **Horário atípico** em relação ao padrão do próprio pagador.
   - **Impossible travel:** duas compras em locais distantes (por exemplo, Brasil e Alemanha) em poucos minutos.
2. Se o score fica abaixo do limite, a transação vira `COMPLETED` e o recebedor é creditado. Se atinge o limite, vira `FRAUD_ALERT` e o valor é **estornado** ao pagador, exatamente uma vez.
3. Cada análise gera um **dossiê de risco** no **MongoDB**, com as regras avaliadas, os pontos, as coordenadas e o score.
4. Uma análise que falha é reprocessada com backoff exponencial. Se continuar falhando, a mensagem vai para uma **DLQ**, de onde é reprocessada manualmente.

```
PRELIMINARY_APPROVED ──► COMPLETED     (análise limpa → recebedor creditado)
                     └─► FRAUD_ALERT   (fraude → valor estornado ao pagador)
```

Os pesos das regras, o limite de fraude e os demais parâmetros são ajustados em tempo de execução por um **Admin**, sem redeploy.

## Decisões de arquitetura

| Decisão | Status |
|---|---|
| Separação entre fast path e slow path, desacoplados via RabbitMQ | Definida (PRD) |
| Idempotência via `Idempotency-Key` no Redis (TTL de 24h; Redis fora do ar → `503`) | Definida (PRD) |
| Estorno em caso de fraude; o recebedor só é creditado em `COMPLETED` | Definida (PRD) |
| JWT emitido por um endpoint de login próprio, com papéis `ACCOUNT_HOLDER` e `ADMIN` | Definida (PRD) |
| Garantia de entrega do evento (transactional outbox ou alternativa) | Na etapa de Architecture |
| Estratégia de lock do saldo (pessimista **ou** otimista: só uma) | Na etapa de Architecture |
| Latência do fast path: meta de p95 < 100 ms | Meta. Será medida com teste de carga, e o número real publicado aqui |

## Tecnologias

- **Linguagem e framework:** Java 21, Spring Boot 3, Spring Data JPA, Spring Data MongoDB, Spring Security (JWT)
- **Persistência:** PostgreSQL 16 (contas e transações), MongoDB 7 (dossiês de risco), Redis 7 (idempotência e cache de regras)
- **Mensageria:** RabbitMQ 3.x
- **Testes:** JUnit 5, Mockito, Testcontainers
- **Infraestrutura e entrega:** Docker Compose, GitHub Actions (CI) e publicação das imagens no GitHub Container Registry a cada tag (CD)

## Como a IA foi usada

O projeto é desenvolvido com o **Claude Code**, guiado pelo **[BMad Method](https://github.com/bmad-code-org/BMAD-METHOD)** (*Breakthrough Method for Agile AI-Driven Development*). Cada etapa gera um artefato versionado em [`_bmad-output/`](_bmad-output/):

| Etapa | Artefato | Status |
|---|---|---|
| PRD | [`prd.md`](_bmad-output/planning-artifacts/prds/prd-payment-risk-gateway-2026-11-25/prd.md) + [`addendum.md`](_bmad-output/planning-artifacts/prds/prd-payment-risk-gateway-2026-11-25/addendum.md) | ✅ |
| Architecture | ADRs e documento de arquitetura | ⬜ |
| Epics & Stories | Stories rastreáveis aos requisitos do PRD | ⬜ |
| Build + Code Review | Uma branch e um PR por story | ⬜ |

As instruções do agente ficam em [`CLAUDE.md`](CLAUDE.md), e o contexto do projeto em [`docs/kickoff.md`](docs/kickoff.md). O histórico de decisões do PRD está em [`.memlog.md`](_bmad-output/planning-artifacts/prds/prd-payment-risk-gateway-2026-11-25/.memlog.md).

**Saídas da IA que foram revisadas e corrigidas** (lista atualizada a cada etapa):
1. **Nome do método.** A primeira versão deste README e do `CLAUDE.md` explicava BMAD como "Breakdown, Model, Architecture, Deliver". Isso estava errado. O nome foi conferido na fonte oficial e corrigido.
2. **Jornada de fraude impossível.** O primeiro rascunho do PRD descrevia um ataque de R$ 2.000 contra uma conta com R$ 420. O pagamento seria rejeitado por falta de saldo e a fraude nunca seria detectada. Uma revisão adversarial pegou o erro antes de virar código.
3. **Falso positivo no histórico.** Sem definir quais transações contam como "histórico", a fraude de Berlim entraria no padrão da vítima, e a próxima compra legítima dela em São Paulo dispararia o alerta. Ficou definido que só transações `COMPLETED` contam.
4. **Promessas sem código.** A versão anterior deste README afirmava, sem nenhum código escrito, latência abaixo de 100 ms, lock pessimista *e* otimista, OAuth2, CQRS-lite e validação estática de segurança. Esses itens foram removidos ou reclassificados como meta.

## Como executar

As instruções serão adicionadas quando a primeira story de infraestrutura for entregue. O objetivo é subir tudo (PostgreSQL, Redis, MongoDB, RabbitMQ, a Payment API e o Fraud Worker) com um único `docker compose up` e validar o sistema com um script de smoke test.

## Próximos passos (fora do MVP)

- Notificação do resultado via webhook. No MVP, o cliente consulta `GET /transactions/{id}`.
- UI em Angular para quem paga e para o Admin.
- Ferramenta de reprocessamento da DLQ. No MVP, o reprocessamento é manual, pelo console do RabbitMQ, com passo a passo documentado aqui.
- Comparação, com benchmark, entre as duas estratégias de lock.
- Listagem do histórico de transações e API de consulta dos dossiês de risco.

## Licença

Distribuído sob a licença MIT. Veja [`LICENSE`](LICENSE).
