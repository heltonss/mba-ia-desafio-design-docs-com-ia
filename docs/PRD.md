# PRD — Sistema de Webhooks de Notificação de Pedidos

Versão: 2.0 (consolidação final)
Data: 24/10/2026
Responsável: Larissa (Tech Lead)
Status: Consolidação aprovada do pacote de design docs

> Este PRD é o documento de mais alto nível do pacote. Ele diz **por quê** a feature existe e **o quê** ela entrega. As decisões arquiteturais estão na [RFC](RFC.md) e nos [ADRs](adrs/README.md); os detalhes de construção (contratos HTTP, matriz de erros, fluxos, integração com o código) estão no [FDD](FDD.md). A origem de cada item na reunião está no [TRACKER](TRACKER.md).

---

## 1. Resumo

O OMS (Node.js, TypeScript, Express, Prisma, MySQL) não notifica clientes quando o status de um pedido muda: os clientes B2B (Atlas Comercial, MaxDistribuição e Nova Cargo) fazem polling em `GET /orders`. Esta feature adiciona **webhooks outbound**: quando o status de um pedido muda, o OMS registra um evento em uma **outbox na mesma transação SQL** e um **worker em processo separado** entrega uma notificação HTTPS assinada (HMAC-SHA256) para os endpoints cadastrados por cada cliente.

A entrega é **at-least-once** (deduplicação no cliente via `X-Event-Id`), com **retry com backoff** (`1m/5m/30m/2h/12h`) e **DLQ** com replay manual restrito a ADMIN. A solução reutiliza o MySQL e os padrões existentes do projeto (AppError, Pino, Zod, error middleware, `requireRole`), sem Redis nem fila externa. Prazo estimado: **três sprints**, incluindo revisão de segurança.

---

## 2. Contexto e problema (por quê)

**Público-alvo**
- Clientes B2B que integram seus sistemas ao OMS: Atlas Comercial, MaxDistribuição e Nova Cargo.
- Usuários autenticados do OMS, incluindo os que representam os clientes B2B.
- Times de operação e administração, que acompanham falhas e executam replay de DLQ.

**Problema**
- Os clientes consultam `GET /orders` periodicamente para detectar mudanças de status: integração lenta e cara, sem garantia de que toda mudança é observada.
- Não existe mecanismo de eventos, fila, webhook ou histórico de entregas no OMS atual — essa lacuna é a feature.
- Risco de negócio concreto: a Atlas Comercial sinalizou que pode migrar para um concorrente se a notificação não for entregue até o fim do trimestre.

**Requisito de negócio**
- "Tempo real" com tolerância de até **10 segundos** entre a mudança de status e a entrega da notificação.
- Somente **outbound**: o OMS envia; os clientes não enviam eventos para o OMS.

**Escopo organizacional**
- A feature é desenvolvida no OMS existente (módulo `src/modules/webhooks`, tabelas novas, processo worker novo). Não cria um novo sistema.

---

## 3. Objetivos e metas

| Objetivo | Métrica | Meta |
| --- | --- | --- |
| Notificar em tempo quase real | Latência entre o commit da mudança de status e a entrega bem-sucedida | **Menor que 10 segundos** |
| Eliminar a dependência de polling | Eventos de status assinados por endpoints ativos registrados na outbox | 100% dos eventos assinados têm tentativa de entrega ou vão para a DLQ |
| Recuperar falhas automaticamente | Tentativas antes da DLQ | 1 tentativa inicial + 5 retentativas (6 chamadas), backoff 1m/5m/30m/2h/12h |
| Dar visibilidade ao cliente | Histórico de entregas por endpoint | Últimas 100 entregas com status, payload, resposta e tempo |
| Proteger a entrega | Requisições entregues assinadas | 100% com HMAC-SHA256 e secret única por endpoint |
| Proteger a receita | Prazo de entrega | Três sprints, com revisão de segurança incluída |

---

## 4. Escopo

### Incluso
- CRUD autenticado de configuração de webhook por customer (criar, listar, editar, remover).
- Outbox no MySQL, worker separado em polling de 2s, retry com backoff e DLQ.
- Secret única por endpoint, gerada pela API e devolvida na criação; rotação com grace period de 24h.
- URL somente HTTPS; limite de payload de 64KB com erro acima do limite.
- Histórico das últimas 100 entregas; replay de DLQ restrito a ADMIN e auditado.
- Reuso dos padrões existentes: `AppError`, Pino, Zod, error middleware, `requireRole`, estrutura de módulos, códigos de erro `WEBHOOK_*`.

### Fora de escopo (descartado ou adiado na reunião)
- **Webhooks inbound** (clientes enviando eventos para o OMS).
- **Envio síncrono** do webhook dentro da transação de mudança de status.
- **Redis Streams / fila externa** (nenhuma infraestrutura nova de mensageria).
- **Trigger do banco** como mecanismo de notificação do worker.
- **Garantia exactly-once**.
- **Retry com 3 tentativas** ou **retry infinito**.
- **Fallback por email** para webhook falhando (adiado para próxima fase).
- **Rate limiting de saída** (observar e decidir depois).
- **Dashboard visual** para o cliente.
- **Arquivamento** de eventos entregues após 30 dias.
- **Múltiplos workers** e garantia de ordering global.

As alternativas descartadas estão detalhadas com os trade-offs na [RFC — Alternativas consideradas](RFC.md#alternativas-consideradas).

---

## 5. Requisitos funcionais

Cada FR declara o comportamento esperado e aponta para onde o detalhe está especificado (fluxos e contratos no FDD; decisão no ADR).

### FR-001 — Cadastrar webhook
Usuário autenticado cadastra um endpoint para um customer, com URL HTTPS e lista de status assinados. A secret é **gerada pela API e devolvida apenas na resposta de criação**.
- **Regras:** `customerId` explícito no body/path (não vem do JWT); URL `http` ou inválida é rejeitada; status inválidos são rejeitados.
- Detalhe: [FDD §4 e §5 — `POST /api/v1/webhooks`](FDD.md).

### FR-002 — Listar webhooks de um customer
Usuário autenticado lista os webhooks configurados de um customer, **sem expor a secret** na listagem.
- Detalhe: FDD §5.

### FR-003 — Editar webhook
Usuário autenticado altera URL, status assinados e estado ativo. A secret **não é editável** por este endpoint (a rotação tem fluxo próprio).
- Detalhe: FDD §5 — `PATCH /api/v1/webhooks/:id`.

### FR-004 — Remover webhook
Usuário autenticado remove um webhook.
- Detalhe: FDD §5 — `DELETE /api/v1/webhooks/:id`.

### FR-005 — Rotacionar secret
Usuário autenticado gera nova secret para um webhook; a anterior **permanece válida por 24 horas** (grace period) e depois expira. Caminho do endpoint proposto no FDD.
- Detalhe: FDD §5 — `POST /api/v1/webhooks/:id/rotate-secret`; decisão no [ADR-004](adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md).

### FR-006 — Registrar evento na outbox na mudança de status
Toda mudança de status elegível insere um evento na `webhook_outbox` **dentro da mesma transação** do `changeStatus`; se a inserção falhar, a transação inteira reverte (status, histórico, estoque). O payload é um **snapshot renderizado na inserção** (com `event_id` UUID), filtrado pelos status que cada webhook do customer assina: se nenhum webhook assina o novo status, nada é inserido.
- Decisão: [ADR-001](adrs/ADR-001-outbox-transacional-no-mysql.md). Integração: FDD §"Integração com o sistema existente".

### FR-007 — Entregar webhook pelo worker
O worker (processo separado, polling de 2s) envia `POST` JSON para o endpoint com `Content-Type: application/json` e os headers `X-Signature` (HMAC-SHA256 sobre o corpo), `X-Timestamp`, `X-Webhook-Id` e `X-Event-Id`. Timeout HTTP de **10 segundos**. Resposta 2xx marca o evento como entregue; timeout, erro de rede ou 5xx encaminham para retry. **A política de retry para 4xx está pendente** (ver Questões em aberto).
- Decisões: [ADR-002](adrs/ADR-002-worker-em-processo-separado-com-polling.md), [ADR-005](adrs/ADR-005-entrega-at-least-once-com-x-event-id.md). Detalhe: FDD §4.

### FR-008 — Aplicar retry com backoff e mover para a DLQ
Falhas retryáveis (5xx, timeout, erro de rede) seguem 1 tentativa inicial + 5 retentativas com backoff `1m/5m/30m/2h/12h` (~15h de janela). Esgotadas as tentativas, o evento vai para a tabela `webhook_dead_letter` com payload, motivo e timestamp, e sai da fila ativa da outbox.
- Decisão: [ADR-003](adrs/ADR-003-retry-com-backoff-e-dlq.md). Detalhe: FDD §4 e §6.

### FR-009 — Consultar histórico de entregas
Usuário autenticado consulta as **últimas 100 entregas** de um webhook, com sucesso/falha, payload, resposta e tempo de resposta.
- Detalhe: FDD §5 — `GET /api/v1/webhooks/:id/deliveries`.

### FR-010 — Reprocessar evento da DLQ
Usuário com **role ADMIN** recoloca um evento da DLQ na outbox como pendente, com **registro de auditoria** (quem, quando, qual evento). Qualquer outro role recebe 403.
- Detalhe: FDD §5 — `POST /api/v1/admin/webhooks/dead-letter/:id/replay`.

### Regras transversais
- Autenticação JWT Bearer obrigatória em todos os endpoints da feature.
- CRUD de configuração e consulta de entregas: **qualquer role autenticada** nesta fase; replay de DLQ: **ADMIN** (via `requireRole`).

---

## 6. Requisitos não funcionais (resumo)

| NFR | Requisito |
| --- | --- |
| Latência | Entrega abaixo de **10 segundos** (polling de 2s + timeout HTTP de 10s por chamada) |
| Tamanho | Payload máximo de **64KB**, com erro (sem truncamento) acima do limite |
| Entrega | **At-least-once** com deduplicação no cliente por `X-Event-Id` (UUID estável entre retries e replay) |
| Integridade | Payload é **snapshot** renderizado no momento da inserção; ordenação por `order_id` enquanto houver um único worker (sem ordering global) |
| Segurança | HMAC-SHA256 sobre o corpo, secret única por endpoint, URL somente HTTPS, rotação com grace period de 24h, auditoria do replay |
| Padrões | Reuso de `AppError`, Pino, Zod, error middleware e `requireRole`; códigos `WEBHOOK_*` |
| Observabilidade | Logs estruturados Pino, métricas e tracing — detalhe no [FDD §7](FDD.md#7-observabilidade) |

---

## 7. Decisões e trade-offs (consolidação)

Cada decisão tem ADR próprio com contexto, alternativas e consequências completas. Aqui, apenas o índice consolidado:

| Decisão | Resumo | ADR |
| --- | --- | --- |
| Outbox transacional no MySQL | Evento gravado na mesma transação do `changeStatus`; sem infraestrutura de mensageria nova | [ADR-001](adrs/ADR-001-outbox-transacional-no-mysql.md) |
| Worker em processo separado com polling | Polling a cada 2s, single-worker, ordenação implícita por `order_id` | [ADR-002](adrs/ADR-002-worker-em-processo-separado-com-polling.md) |
| Retry com backoff e DLQ | 5 retentativas `1m/5m/30m/2h/12h`; DLQ em tabela separada; replay ADMIN auditado | [ADR-003](adrs/ADR-003-retry-com-backoff-e-dlq.md) |
| HMAC-SHA256 com secret por endpoint | Secret única gerada pela API, rotação com grace period de 24h, URL somente HTTPS, payload ≤64KB | [ADR-004](adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md) |
| Entrega at-least-once com `X-Event-Id` | UUID único por evento para deduplicação no cliente | [ADR-005](adrs/ADR-005-entrega-at-least-once-com-x-event-id.md) |
| Snapshot na inserção e filtro de status | Payload renderizado na inserção; eventos filtrados pelos status assinados | [ADR-001](adrs/ADR-001-outbox-transacional-no-mysql.md) |
| Reuso dos padrões existentes | Módulo `src/modules/webhooks`, AppError, Pino, error middleware, `requireRole` | [ADR-006](adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md) |

A visão arquitetural consolidada (proposta, alternativas descartadas, impacto e riscos) está na [RFC](RFC.md).

---

## 8. Questões em aberto

Pontos não fechados na reunião, listados e qualificados na [RFC — Questões em aberto](RFC.md#questões-em-aberto): rate limiting de saída, fallback por email, formato de convivência das duas secrets na rotação, política de retry para 4xx, recuperação de eventos presos em `PROCESSANDO`, ordenação por `order_id` vs. retry, arquivamento após ~30 dias e escala para múltiplos workers. Nenhum deles bloqueia o início da implementação, mas o formato de convivência das secrets e a política de retry para 4xx devem ser fechados com a Sofia antes do FDD final.

---

## 9. Dependências

- **Externa:** endpoints HTTPS dos clientes B2B, com validação de assinatura HMAC e deduplicação por `X-Event-Id` implementados do lado deles.
- **Organizacional:** revisão de segurança da Sofia (2 dias úteis antes do deploy, cobrindo HMAC e geração de secret).
- **Organizacional:** documentação no portal do desenvolvedor pelo Marcos (payload, headers, HMAC, deduplicação, retry, DLQ, rotação).
- **Técnica:** MySQL e Prisma existentes (worker usa a mesma `DATABASE_URL` com PrismaClient separado por processo).
- **Técnica:** componentes e padrões existentes do OMS (AppError, Pino, error middleware, Zod, `requireRole`, máquina de estados de pedidos).
- **Técnica:** novas tabelas (`webhook_outbox`, `webhook_dead_letter`, configuração, entregas) migradas antes da entrada do worker em produção.

---

## 10. Riscos e mitigação

Probabilidades e impactos são estimativas iniciais (hipótese, não quantificadas na reunião). Mitigações detalhadas na [RFC — Impacto e riscos](RFC.md#impacto-e-riscos) e no [FDD §10](FDD.md#10-riscos-e-mitigação).

| Risco | Probabilidade | Impacto | Mitigação |
| --- | --- | --- | --- |
| Cliente indisponível ou lento durante a entrega | Alta | Eventos atrasam e, após a janela, só saem com replay manual | Retry 1+5 com backoff ~15h, timeout de 10s, DLQ com replay ADMIN |
| Vazamento de secret ou comprometimento do endpoint do cliente | Média | Terceiros podem forjar requisições ou ler dados de pedidos | Secret única por endpoint, HMAC-SHA256, HTTPS obrigatório, rotação 24h, payload enxuto |
| Divergência entre mudança de status e registro na outbox | Baixa | Cliente não recebe notificação que deveria receber | Inserção na mesma transação com rollback total; testes de rollback |
| Acúmulo de eventos na outbox e latência acima da meta | Média | Entregas ultrapassam 10s | Índices em status/`created_at`, lotes pequenos, métricas de backlog, single-worker |
| Duplicidade de entrega não tratada pelo cliente | Média | Efeitos duplicados no sistema do cliente | `X-Event-Id` estável, documentação de deduplicação no portal |
| Perda de clientes B2B por atraso na entrega | Média | Atlas pode migrar para concorrente | Entrega em 3 sprints com revisão de segurança, comunicação pelo Marcos |

---

## 11. Critérios de aceitação (alto nível)

- O CRUD de webhooks está disponível sob `/api/v1` com autenticação JWT; a criação devolve a secret apenas uma vez.
- A URL é validada como HTTPS; cada endpoint tem secret única e rotação com a anterior válida por 24h.
- A mudança de status insere o evento na outbox **na mesma transação**; falha na inserção reverte tudo; status não assinado não gera evento.
- O worker roda em processo separado com polling de 2s; entregas usam os headers `X-Signature`, `X-Timestamp`, `X-Webhook-Id`, `X-Event-Id`; timeout de 10s.
- Falhas retryáveis seguem 1+5 retentativas (6 chamadas) com `1m/5m/30m/2h/12h`; esgotadas, o evento vai para a DLQ com payload, motivo e timestamp.
- O replay de DLQ exige role ADMIN e registra quem executou.
- O histórico retorna as últimas 100 entregas; payloads acima de 64KB são rejeitados sem truncamento.
- A feature é entregue em **três sprints**, incluindo a revisão de segurança da Sofia.

Critérios de aceite técnicos verificáveis (migração, testes, rotas): [FDD §9](FDD.md#9-critérios-de-aceite-técnicos).

---

## 12. Testes e validação

Estratégia resumida; o detalhamento operacional está no FDD:
- Unitários: máquina de estados, filtro de status, payload, backoff, HMAC, geração de secret.
- Integração: CRUD, autenticação, transação `changeStatus` + outbox, worker com servidor HTTP de teste, retry, DLQ, entregas, replay. Nunca usar banco de produção nos testes.
- Segurança: role ADMIN no replay, rejeição de URL `http`, validação de assinatura, rotação, não exposição de secret.
- QA manual guiado por roteiro e revisão de segurança da Sofia antes do deploy.

---

## 13. Referências do pacote

| Documento | Papel | Altura |
| --- | --- | --- |
| `docs/PRD.md` (este) | Por quê / o quê: problema, metas, escopo, FRs, riscos | Visão de produto |
| `docs/RFC.md` | Arquitetura proposta, alternativas descartadas, questões em aberto, riscos | Decisão arquitetural |
| `docs/adrs/ADR-001..006` | Uma decisão cada, com contexto, alternativas e consequências | Decisão individual |
| `docs/FDD.md` | Contratos HTTP, matriz `WEBHOOK_*`, fluxos, integração com o código, observabilidade, aceite técnico | Como construir |
| `docs/TRACKER.md` | Rastreabilidade de cada item para `TRANSCRICAO.md` ou código real | Rastreabilidade |
