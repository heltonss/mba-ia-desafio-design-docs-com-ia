# Tracker de Rastreabilidade

Referência cruzada entre o pacote de design docs (`PRD`, `RFC`, `FDD`, `ADRs`) e suas origens na `TRANSCRICAO.md` ou no código existente. Cada linha mapeia um item identificável à fonte que o sustenta.

- **Fonte `TRANSCRICAO`**: localização no formato `[hh:mm] Nome`, usando os cinco participantes da reunião.
- **Fonte `CODIGO`**: caminho de arquivo real existente no repositório (arquivos propostos como `src/worker.ts` e `src/modules/webhooks/*` não são usados como fonte).
- Itens de hipótese ou inferência estão marcados no próprio resumo; quando não há origem identificável, não recebem linha (por regra de não alucinação).

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| PRD-FR-01 | docs/PRD.md | Requisito Funcional | Cadastrar webhook (POST) com URL, lista de status e secret gerada pela API e devolvida na criação | TRANSCRICAO | [09:31] Marcos |
| PRD-FR-02 | docs/PRD.md | Requisito Funcional | Listar os webhooks de um customer | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-03 | docs/PRD.md | Requisito Funcional | Editar webhook (PATCH): URL, status assinados e estado ativo | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-04 | docs/PRD.md | Requisito Funcional | Remover webhook (DELETE) | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-05 | docs/PRD.md | Requisito Funcional | Rotacionar secret mantendo a anterior válida por 24 horas | TRANSCRICAO | [09:21] Sofia |
| PRD-FR-06 | docs/PRD.md | Requisito Funcional | Registrar evento na outbox na mesma transação da mudança de status | TRANSCRICAO | [09:40] Bruno |
| PRD-FR-07 | docs/PRD.md | Requisito Funcional | Worker entrega HTTP assinada dos eventos pendentes da outbox | TRANSCRICAO | [09:09] Diego |
| PRD-FR-08 | docs/PRD.md | Requisito Funcional | Aplicar retry com backoff e mover eventos esgotados para a DLQ | TRANSCRICAO | [09:17] Diego |
| PRD-FR-09 | docs/PRD.md | Requisito Funcional | Consultar as últimas 100 entregas de um webhook | TRANSCRICAO | [09:34] Marcos |
| PRD-FR-10 | docs/PRD.md | Requisito Funcional | Replay manual de DLQ restrito a ADMIN e auditado | TRANSCRICAO | [09:18] Diego |
| PRD-RES-01 | docs/PRD.md | Restrição | `customerId` explícito no body/path; não vem do JWT (correção da sugestão inicial) | TRANSCRICAO | [09:32] Larissa |
| PRD-RES-02 | docs/PRD.md | Restrição | CRUD e consulta de entregas: qualquer role autenticada nesta fase; replay exige ADMIN | TRANSCRICAO | [09:37] Sofia |
| PRD-NFR-01 | docs/PRD.md | Requisito Não Funcional | "Tempo real" = latência de entrega abaixo de 10 segundos | TRANSCRICAO | [09:02] Marcos |
| PRD-NFR-02 | docs/PRD.md | Requisito Não Funcional | Payload máximo de 64KB, com erro (sem truncamento) acima do limite | TRANSCRICAO | [09:24] Larissa |
| PRD-NFR-03 | docs/PRD.md | Requisito Não Funcional | Autenticação JWT Bearer em todos os endpoints da feature | CODIGO | src/middlewares/auth.middleware.ts |
| PRD-NFR-04 | docs/PRD.md | Requisito Não Funcional | Assinatura HMAC-SHA256 sobre o corpo com secret única por endpoint | TRANSCRICAO | [09:22] Sofia |
| PRD-NFR-05 | docs/PRD.md | Requisito Não Funcional | URL do webhook somente HTTPS (HTTP rejeitado na validação) | TRANSCRICAO | [09:23] Sofia |
| PRD-NFR-06 | docs/PRD.md | Requisito Não Funcional | Garantia at-least-once com deduplicação no cliente por `X-Event-Id` | TRANSCRICAO | [09:26] Larissa |
| PRD-NFR-07 | docs/PRD.md | Requisito Não Funcional | Payload é snapshot renderizado no momento da inserção | TRANSCRICAO | [09:52] Larissa |
| PRD-NFR-08 | docs/PRD.md | Requisito Não Funcional | Ordenação por `order_id` enquanto houver um único worker (sem ordering global) | TRANSCRICAO | [09:12] Diego |
| PRD-NFR-09 | docs/PRD.md | Requisito Não Funcional | Logs estruturados reutilizando o Pino existente | CODIGO | src/shared/logger/index.ts |
| PRD-MET-01 | docs/PRD.md | Métrica | Meta: latência entre commit e entrega menor que 10 segundos | TRANSCRICAO | [09:02] Marcos |
| PRD-MET-02 | docs/PRD.md | Métrica | 1 tentativa inicial + 5 retentativas (6 chamadas) com backoff 1m/5m/30m/2h/12h | TRANSCRICAO | [09:17] Diego |
| PRD-MET-03 | docs/PRD.md | Métrica | Histórico com as últimas 100 entregas (status, payload, resposta, tempo) | TRANSCRICAO | [09:34] Marcos |
| PRD-ESC-IN-01 | docs/PRD.md | Escopo (incluso) | CRUD autenticado de configuração de webhook por customer | TRANSCRICAO | [09:31] Marcos |
| PRD-ESC-IN-02 | docs/PRD.md | Escopo (incluso) | Outbox no MySQL, worker separado em polling de 2s, retry e DLQ | TRANSCRICAO | [09:06] Diego |
| PRD-ESC-IN-03 | docs/PRD.md | Escopo (incluso) | Reuso dos padrões existentes (AppError, Pino, Zod, `requireRole`, módulos) | TRANSCRICAO | [09:30] Larissa |
| PRD-ESC-OUT-01 | docs/PRD.md | Fora de escopo | Webhooks inbound (clientes enviando eventos para o OMS) | TRANSCRICAO | [09:02] Marcos |
| PRD-ESC-OUT-02 | docs/PRD.md | Fora de escopo | Envio síncrono dentro da transação de mudança de status | TRANSCRICAO | [09:04] Bruno |
| PRD-ESC-OUT-03 | docs/PRD.md | Fora de escopo | Redis Streams/fila externa (nova infraestrutura de mensageria) | TRANSCRICAO | [09:07] Diego |
| PRD-ESC-OUT-04 | docs/PRD.md | Fora de escopo | Trigger do banco como mecanismo de notificação do worker | TRANSCRICAO | [09:09] Diego |
| PRD-ESC-OUT-05 | docs/PRD.md | Fora de escopo | Garantia exactly-once | TRANSCRICAO | [09:25] Diego |
| PRD-ESC-OUT-06 | docs/PRD.md | Fora de escopo | Retry com 3 tentativas ou retry infinito | TRANSCRICAO | [09:16] Diego |
| PRD-ESC-OUT-07 | docs/PRD.md | Fora de escopo | Fallback por email para webhook falhando (fase futura) | TRANSCRICAO | [09:37] Larissa |
| PRD-ESC-OUT-08 | docs/PRD.md | Fora de escopo | Rate limiting de saída (observar e decidir depois) | TRANSCRICAO | [09:39] Larissa |
| PRD-ESC-OUT-09 | docs/PRD.md | Fora de escopo | Dashboard visual para o cliente | TRANSCRICAO | [09:40] Larissa |
| PRD-ESC-OUT-10 | docs/PRD.md | Fora de escopo | Arquivamento de eventos entregues após 30 dias | TRANSCRICAO | [09:08] Diego |
| PRD-DEC-01 | docs/PRD.md | Decisão / Trade-off | Outbox no MySQL em vez de envio síncrono ou fila externa | TRANSCRICAO | [09:06] Diego |
| PRD-DEC-02 | docs/PRD.md | Decisão / Trade-off | Worker separado em polling de 2 segundos | TRANSCRICAO | [09:10] Larissa |
| PRD-DEC-03 | docs/PRD.md | Decisão / Trade-off | Retry com 5 retentativas e DLQ em tabela separada | TRANSCRICAO | [09:18] Diego |
| PRD-DEC-04 | docs/PRD.md | Decisão / Trade-off | HMAC-SHA256 com secret única por endpoint e rotação 24h | TRANSCRICAO | [09:22] Sofia |
| PRD-DEC-05 | docs/PRD.md | Decisão / Trade-off | At-least-once com deduplicação por `X-Event-Id` | TRANSCRICAO | [09:26] Larissa |
| PRD-DEC-06 | docs/PRD.md | Decisão / Trade-off | Snapshot na inserção e filtro de status na outbox | TRANSCRICAO | [09:52] Diego |
| PRD-DEC-07 | docs/PRD.md | Decisão / Trade-off | Reutilizar os padrões existentes do projeto | TRANSCRICAO | [09:30] Larissa |
| PRD-RISK-01 | docs/PRD.md | Risco | Cliente indisponível/lento durante a entrega (probabilidade/impacto são hipótese) | TRANSCRICAO | [09:15] Diego |
| PRD-RISK-02 | docs/PRD.md | Risco | Vazamento de secret ou comprometimento do endpoint do cliente | TRANSCRICAO | [09:22] Diego |
| PRD-RISK-03 | docs/PRD.md | Risco | Divergência entre mudança de status e registro na outbox | TRANSCRICAO | [09:41] Diego |
| PRD-RISK-04 | docs/PRD.md | Risco | Acúmulo de eventos na outbox e latência acima da meta | TRANSCRICAO | [09:08] Diego |
| PRD-RISK-05 | docs/PRD.md | Risco | Duplicidade de entrega não tratada pelo cliente | TRANSCRICAO | [09:25] Diego |
| PRD-RISK-06 | docs/PRD.md | Risco | Perda de clientes B2B por atraso (Atlas ameaçou migrar) | TRANSCRICAO | [09:00] Marcos |
| PRD-DEP-01 | docs/PRD.md | Dependência | Endpoints HTTPS dos clientes B2B + validação HMAC e dedup no cliente | TRANSCRICAO | [09:20] Sofia |
| PRD-DEP-02 | docs/PRD.md | Dependência | Revisão de segurança da Sofia (2 dias úteis antes do deploy) | TRANSCRICAO | [09:46] Sofia |
| PRD-DEP-03 | docs/PRD.md | Dependência | Documentação no portal do desenvolvedor pelo Marcos | TRANSCRICAO | [09:26] Marcos |
| PRD-DEP-04 | docs/PRD.md | Dependência | MySQL existente e Prisma (worker usa a mesma `DATABASE_URL`, PrismaClient separado) | TRANSCRICAO | [09:30] Bruno |
| PRD-DEP-05 | docs/PRD.md | Dependência | Novas tabelas e migração Prisma antes do worker em produção | TRANSCRICAO | [09:06] Diego |
| PRD-ACE-01 | docs/PRD.md | Critério de aceite | Mudança de status insere o evento na outbox na mesma transação; falha reverte tudo | TRANSCRICAO | [09:40] Bruno |
| PRD-ACE-02 | docs/PRD.md | Critério de aceite | Replay de DLQ exige role ADMIN e registra quem executou | TRANSCRICAO | [09:36] Sofia |
| PRD-ACE-03 | docs/PRD.md | Critério de aceite | Prazo de entrega em três sprints, incluindo revisão de segurança | TRANSCRICAO | [09:47] Larissa |
| RFC-TLDR-01 | docs/RFC.md | Resumo | Proposta: outbox transacional + worker separado + HMAC-SHA256 + retry/DLQ | TRANSCRICAO | [09:48] Larissa |
| RFC-CONTEXTO-01 | docs/RFC.md | Contexto | Clientes B2B fazem polling em `GET /orders`; Atlas pode migrar no fim do trimestre | TRANSCRICAO | [09:00] Marcos |
| RFC-ALT-01 | docs/RFC.md | Alternativa descartada | Envio síncrono dentro de `changeStatus` | TRANSCRICAO | [09:04] Bruno |
| RFC-ALT-02 | docs/RFC.md | Alternativa descartada | Redis Streams / fila externa (Redis Cluster) | TRANSCRICAO | [09:07] Diego |
| RFC-ALT-03 | docs/RFC.md | Alternativa descartada | Trigger do banco como notificador | TRANSCRICAO | [09:09] Diego |
| RFC-ALT-04 | docs/RFC.md | Alternativa descartada | Retry com 3 tentativas | TRANSCRICAO | [09:16] Diego |
| RFC-ALT-05 | docs/RFC.md | Alternativa descartada | Retry infinito com backoff | TRANSCRICAO | [09:15] Diego |
| RFC-ALT-06 | docs/RFC.md | Alternativa descartada | Entrega exactly-once | TRANSCRICAO | [09:25] Diego |
| RFC-OQ-01 | docs/RFC.md | Questão em aberto | Rate limiting de saída: observar e decidir depois | TRANSCRICAO | [09:39] Larissa |
| RFC-OQ-02 | docs/RFC.md | Questão em aberto | Fallback por email para webhook com falha (adiado) | TRANSCRICAO | [09:37] Larissa |
| RFC-OQ-03 | docs/RFC.md | Questão em aberto | Formato de convivência das duas secrets durante a rotação (24h) | TRANSCRICAO | [09:22] Sofia |
| RFC-OQ-04 | docs/RFC.md | Questão em aberto | Política de retry para resposta 4xx do cliente | TRANSCRICAO | [09:42] Diego |
| RFC-OQ-05 | docs/RFC.md | Questão em aberto | Ordenação por `order_id` vs. retry atrasado | TRANSCRICAO | [09:12] Diego |
| RFC-RISK-01 | docs/RFC.md | Risco | Worker único é ponto único de processamento | TRANSCRICAO | [09:11] Diego |
| RFC-RISK-02 | docs/RFC.md | Risco | Inserção na outbox acoplada ao `changeStatus` pode bloquear a operação | TRANSCRICAO | [09:41] Diego |
| RFC-RISK-03 | docs/RFC.md | Risco | Cliente indisponível por mais de ~15h depende de replay manual | TRANSCRICAO | [09:17] Marcos |
| RFC-REL-01 | docs/RFC.md | Decisão relacionada | Link para ADR-001 (outbox no MySQL) | TRANSCRICAO | [09:06] Diego |
| RFC-REL-02 | docs/RFC.md | Decisão relacionada | Link para ADR-004 (HMAC-SHA256 secret por endpoint) | TRANSCRICAO | [09:20] Sofia |
| FDD-CONTRATO-01 | docs/FDD.md | Contrato (endpoint) | `POST /api/v1/webhooks` cria webhook e devolve a secret | TRANSCRICAO | [09:31] Marcos |
| FDD-CONTRATO-02 | docs/FDD.md | Contrato (endpoint) | Listar webhooks de um customer (GET) | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-03 | docs/FDD.md | Contrato (endpoint) | `PATCH /api/v1/webhooks/:id` edita webhook | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-04 | docs/FDD.md | Contrato (endpoint) | `DELETE /api/v1/webhooks/:id` remove webhook | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-05 | docs/FDD.md | Contrato (endpoint) | Endpoint de rotação de secret (caminho proposto no FDD) | TRANSCRICAO | [09:21] Sofia |
| FDD-CONTRATO-06 | docs/FDD.md | Contrato (endpoint) | `GET /api/v1/webhooks/:id/deliveries` (últimas 100 entregas) | TRANSCRICAO | [09:34] Marcos |
| FDD-CONTRATO-07 | docs/FDD.md | Contrato (endpoint) | `POST /api/v1/admin/webhooks/dead-letter/:id/replay` (ADMIN) | TRANSCRICAO | [09:18] Diego |
| FDD-CONTRATO-08 | docs/FDD.md | Contrato (função interna) | `publishWebhookEvent(tx, order, fromStatus, toStatus)` recebendo o `tx` | TRANSCRICAO | [09:41] Bruno |
| FDD-CONTRATO-09 | docs/FDD.md | Contrato (entrega outbound) | Payload JSON e headers `X-Signature`, `X-Timestamp`, `X-Webhook-Id`, `X-Event-Id` | TRANSCRICAO | [09:44] Diego |
| FDD-CONTRATO-10 | docs/FDD.md | Contrato (payload) | Campos `event_id`, `event_type=order.status_changed`, timestamp ISO 8601, `order_id`, `order_number`, `from_status`, `to_status`, `customer_id`, `total_cents` (sem itens) | TRANSCRICAO | [09:43] Diego |
| FDD-ERRO-01 | docs/FDD.md | Erro | Prefixo `WEBHOOK_` para códigos de erro do módulo | TRANSCRICAO | [09:29] Larissa |
| FDD-ERRO-02 | docs/FDD.md | Erro | `WEBHOOK_NOT_FOUND` (exemplo citado) | TRANSCRICAO | [09:28] Bruno |
| FDD-ERRO-03 | docs/FDD.md | Erro | `WEBHOOK_INVALID_URL` (URL não HTTPS) | TRANSCRICAO | [09:23] Sofia |
| FDD-ERRO-04 | docs/FDD.md | Erro | `WEBHOOK_SECRET_REQUIRED` (exemplo citado) | TRANSCRICAO | [09:28] Bruno |
| FDD-ERRO-05 | docs/FDD.md | Erro | `WEBHOOK_PAYLOAD_TOO_LARGE` (acima de 64KB) | TRANSCRICAO | [09:24] Larissa |
| FDD-ERRO-06 | docs/FDD.md | Erro | Erros transversais `UNAUTHORIZED`/`FORBIDDEN` via `authenticate`/`requireRole` | CODIGO | src/middlewares/auth.middleware.ts |
| FDD-ERRO-07 | docs/FDD.md | Erro | Envelope de erro `{ code, message, details }` tratado pelo error middleware | CODIGO | src/middlewares/error.middleware.ts |
| FDD-FLOW-01 | docs/FDD.md | Fluxo | Inserção na outbox dentro da transação de `changeStatus`; rollback em falha | TRANSCRICAO | [09:40] Bruno |
| FDD-FLOW-02 | docs/FDD.md | Fluxo | Filtro de status aplicado na inserção (nenhum evento se nenhum webhook assina) | TRANSCRICAO | [09:34] Bruno |
| FDD-FLOW-03 | docs/FDD.md | Fluxo | Worker em polling de 2s processa os pendentes mais antigos em lote pequeno | TRANSCRICAO | [09:09] Diego |
| FDD-FLOW-04 | docs/FDD.md | Fluxo | Timeout HTTP de 10 segundos; 2xx entrega, 5xx/timeout/erro de rede retry | TRANSCRICAO | [09:42] Diego |
| FDD-FLOW-05 | docs/FDD.md | Fluxo | Backoff 1m/5m/30m/2h/12h, ~15h até esgotar e mover para DLQ | TRANSCRICAO | [09:17] Diego |
| FDD-FLOW-06 | docs/FDD.md | Fluxo | DLQ em tabela separada com payload, motivo da falha e timestamp | TRANSCRICAO | [09:18] Diego |
| FDD-FLOW-07 | docs/FDD.md | Fluxo | Replay recoloca o evento na outbox como pendente (preservando `event_id`) | TRANSCRICAO | [09:18] Diego |
| FDD-INT-01 | docs/FDD.md | Integração (código) | Transação de `changeStatus` começa na linha 126 e abre `$transaction` na 131 | CODIGO | src/modules/orders/order.service.ts |
| FDD-INT-02 | docs/FDD.md | Integração (código) | `create` (linha 50) não é alterado e não gera evento | CODIGO | src/modules/orders/order.service.ts |
| FDD-INT-03 | docs/FDD.md | Integração (código) | Máquina de estados: transições válidas, débito em `PENDING->PAID` e reposição em `PAID/PROCESSING->CANCELLED` | CODIGO | src/modules/orders/order.status.ts |
| FDD-INT-04 | docs/FDD.md | Integração (código) | Classes de erro específicas que estendem `AppError` com códigos estáveis | CODIGO | src/shared/errors/http-errors.ts |
| FDD-INT-05 | docs/FDD.md | Integração (código) | `AppError` com `statusCode`, `errorCode` e `details` | CODIGO | src/shared/errors/app-error.ts |
| FDD-INT-06 | docs/FDD.md | Integração (código) | `authenticate` (linha 27) e `requireRole` (linha 49) | CODIGO | src/middlewares/auth.middleware.ts |
| FDD-INT-07 | docs/FDD.md | Integração (código) | Error middleware trata `AppError`, `ZodError` e erros Prisma (P2002/P2025) | CODIGO | src/middlewares/error.middleware.ts |
| FDD-INT-08 | docs/FDD.md | Integração (código) | Logger Pino com `redactPaths` (linha 4) e `createLogger` (linha 13) | CODIGO | src/shared/logger/index.ts |
| FDD-INT-09 | docs/FDD.md | Integração (código) | Montagem das rotas via `buildApiRouter` (linha 21) | CODIGO | src/routes/index.ts |
| FDD-INT-10 | docs/FDD.md | Integração (código) | Prefixo `/api/v1` montado na linha 67 e `express.json` (1mb) na 59 | CODIGO | src/app.ts |
| FDD-INT-11 | docs/FDD.md | Integração (código) | `createPrismaClient()` para instância própria do worker (linha 4) | CODIGO | src/config/database.ts |
| FDD-INT-12 | docs/FDD.md | Integração (código) | Middleware `validate` aplica schemas Zod e converte em `VALIDATION_ERROR` | CODIGO | src/middlewares/validate.middleware.ts |
| FDD-INT-13 | docs/FDD.md | Integração (código) | Helper `paginated` para listagens (linha 22) | CODIGO | src/shared/http/response.ts |
| FDD-INT-14 | docs/FDD.md | Integração (código) | Schema atual não tem tabelas de webhook, outbox ou DLQ (lacuna da feature) | CODIGO | prisma/schema.prisma |
| FDD-INT-15 | docs/FDD.md | Integração (código) | Enum `OrderStatus` (PENDING, PAID, PROCESSING, SHIPPED, DELIVERED, CANCELLED) | CODIGO | prisma/schema.prisma |
| ADR-001 | docs/adrs/ADR-001-outbox-transacional-no-mysql.md | Decisão | Outbox transacional no MySQL, gravada na mesma transação da mudança de status | TRANSCRICAO | [09:06] Diego |
| ADR-001-REF | docs/adrs/ADR-001-outbox-transacional-no-mysql.md | Referência de código | Transação de `changeStatus` e máquina de estados citadas no ADR | CODIGO | src/modules/orders/order.service.ts |
| ADR-002 | docs/adrs/ADR-002-worker-em-processo-separado-com-polling.md | Decisão | Worker em processo separado com polling de 2 segundos, single-worker | TRANSCRICAO | [09:09] Diego |
| ADR-003 | docs/adrs/ADR-003-retry-com-backoff-e-dlq.md | Decisão | Retry 1m/5m/30m/2h/12h (5 retentativas) e DLQ em tabela separada | TRANSCRICAO | [09:17] Diego |
| ADR-004 | docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md | Decisão | HMAC-SHA256 sobre o corpo, secret única por endpoint, rotação com 24h | TRANSCRICAO | [09:20] Sofia |
| ADR-005 | docs/adrs/ADR-005-entrega-at-least-once-com-x-event-id.md | Decisão | Entrega at-least-once com `X-Event-Id` UUID para deduplicação no cliente | TRANSCRICAO | [09:25] Diego |
| ADR-006 | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Decisão | Reuso de `AppError`, Pino, error middleware, `requireRole` e padrão de módulos | TRANSCRICAO | [09:30] Larissa |
| ADR-006-REF | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Referência de código | Padrões de erro, auth, logger e rotas citados no ADR | CODIGO | src/shared/errors/app-error.ts |
| PRD-CONTEXTO-01 | docs/PRD.md | Contexto | Três clientes B2B: Atlas Comercial, MaxDistribuição e Nova Cargo | TRANSCRICAO | [09:00] Marcos |
| PRD-CONTEXTO-02 | docs/PRD.md | Contexto | Transação atual é pesada: atualiza orders, insere history e decrementa estoque | TRANSCRICAO | [09:04] Bruno |
| RFC-PROPOSTA-01 | docs/RFC.md | Proposta | Worker como entrypoint `src/worker.ts` e script `npm run worker` | TRANSCRICAO | [09:11] Larissa |
| RFC-PROPOSTA-02 | docs/RFC.md | Proposta | Entrega estimada em três sprints com revisão de segurança | TRANSCRICAO | [09:47] Larissa |
