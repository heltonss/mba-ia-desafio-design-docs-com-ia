# AGENTS.md

## Propósito do repositório

- Desafio de MBA exclusivamente documental: transformar `TRANSCRICAO.md` e o código OMS existente em um pacote de design docs. **Não** modifique `src/`, `prisma/`, `tests/`, configurações de ferramentas nem `TRANSCRICAO.md`.
- Entregáveis: `docs/PRD.md`, `docs/RFC.md`, `docs/FDD.md`, `docs/TRACKER.md`, `docs/adrs/ADR-NNN-*.md` e um `README.md` da raiz reescrito.
- No momento em que este arquivo foi escrito, o `README.md` da raiz é a especificação do desafio. A entrega final o substitui pelo relato do processo (sobre o desafio, ferramentas, workflow, prompts, iterações, como navegar).
- Escreva todos os documentos em português do Brasil, acompanhando o README e a transcrição.
- Mantenha a altura de cada documento: PRD = por quê/o quê, RFC = arquitetura/proposta, ADRs = uma decisão cada, FDD = detalhe de implementação. Evite duplicar conteúdo entre RFC e FDD.

## Fontes de verdade

- `TRANSCRICAO.md` tem 155 falas no formato `[hh:mm] Nome`. Use-a para requisitos, decisões, alternativas e adiamentos; não invente fatos de produto ou arquitetura.
- Pontos reais de integração com o código:
  - `src/modules/orders/order.service.ts`: a transação de `changeStatus` começa na linha 126; a inserção na outbox deve acontecer dentro dela. `create` começa na linha 50.
  - `src/modules/orders/order.status.ts`: máquina de estados; debita estoque em `PENDING -> PAID` e repõe em `PAID/PROCESSING -> CANCELLED`.
  - `src/shared/errors/app-error.ts` e `http-errors.ts`: `AppError` mais classes específicas com códigos de erro estáveis.
  - `src/middlewares/auth.middleware.ts`: `authenticate` e `requireRole`.
  - `src/middlewares/error.middleware.ts`, `src/shared/logger/index.ts`, `src/routes/index.ts`, `src/app.ts`: tratamento centralizado de erros, Pino, montagem das rotas, prefixo `/api/v1`.
  - `prisma/schema.prisma`: modelos atuais; não existem tabelas de webhook, outbox ou DLQ.
- O app existente não tem eventos, fila nem código de webhook; essa lacuna é a feature.
- `src/worker.ts`, `src/modules/webhooks/` e `npm run worker` são propostos na reunião, mas ainda **não existem**. Marque-os como novos nos docs; cite apenas caminhos existentes como código atual.

## Decisões fechadas (da transcrição; não contradiga)

- Outbox no MySQL gravada atomicamente com a mudança de status; worker em processo separado, em polling a cada 2s; apenas um worker, então a ordenação é por `order_id`, não global.
- 5 tentativas com backoff `1m/5m/30m/2h/12h`, depois uma tabela separada `webhook_dead_letter`; o endpoint de replay manual é restrito a ADMIN e auditado.
- HMAC-SHA256 sobre o corpo da requisição, secret gerada pela API e única por endpoint (devolvida na criação), grace period de 24h para rotação; URL somente HTTPS; limite de payload de 64KB (erro acima disso).
- Entrega at-least-once com `X-Event-Id` UUID para deduplicação no cliente. Headers: `X-Signature`, `X-Timestamp`, `X-Webhook-Id`, `Content-Type: application/json`; timeout HTTP de 10s.
- Eventos são filtrados na inserção da outbox pelos status assinados pelo endpoint; IDs da outbox são UUIDs; o payload é um snapshot renderizado na inserção.
- Formato da API: CRUD autenticado em `/webhooks`, `GET /webhooks/:id/deliveries` (últimas 100) e `POST /admin/webhooks/dead-letter/:id/replay` restrito a ADMIN. `customerId` é explícito no body/path, não vem do JWT (a transcrição corrige a sugestão inicial).
- Siga o padrão de módulo existente para o novo módulo `src/modules/webhooks` e reutilize `AppError`, Pino, Zod, o error middleware e códigos de erro `WEBHOOK_*`.
- Explicitamente rejeitados/adiados: envio síncrono, Redis Streams/fila, trigger do banco como notificador, entrega exactly-once, retry com 3 tentativas ou infinito, webhooks inbound, fallback por email, rate limiting de saída (observar depois), dashboard para cliente, arquivamento após 30 dias.

## Regras de aceite que agentes costumam esquecer

- Arquivos de ADR devem ser `docs/adrs/ADR-NNN-titulo-em-kebab-case.md`. O exemplo `0001-...` em `docs/adrs/README.md` conflita com o README da raiz; siga o README da raiz. Produza 5-8 ADRs cobrindo pelo menos 5 das 6 decisões principais. Cada ADR precisa de Status, Contexto, Decisão, Alternativas Consideradas, Consequências; pelo menos um deve referenciar código real.
- As colunas de `docs/TRACKER.md` são exatamente: `| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |`. `Fonte` é apenas `TRANSCRICAO` ou `CODIGO`; localizações da transcrição usam `[hh:mm] Nome`; localizações de código devem ser caminhos reais existentes. Não use arquivos propostos (`src/worker.ts`, `src/modules/webhooks/*`) como fontes `CODIGO`.
- Mínimos do tracker: pelo menos 80% de cobertura, pelo menos 70% das linhas com fonte na transcrição e pelo menos 5 linhas de código.
- FDD precisa de pelo menos 4 endpoints HTTP com exemplos de request/response e status codes, uma matriz de erros `WEBHOOK_*`, pelo menos 4 caminhos reais em "Integração com o sistema existente" e métricas + logs + tracing.
- PRD precisa de pelo menos 8 requisitos funcionais, uma meta quantificada, dois itens explicitamente descartados/adiados e dois riscos com probabilidade/impacto/mitigação.
- RFC deve ter 2-4 páginas, listar pelo menos 2 alternativas descartadas e 2 questões em aberto, linkar pelo menos 2 ADRs e usar os cinco participantes da reunião como revisores.
- README do processo precisa de pelo menos 1 ferramenta de IA, 2 prompts customizados em blocos de código e 2 iterações concretas.
- No momento em que este arquivo foi escrito, `docs/*.md` contêm apenas títulos/comentários; substitua o conteúdo em vez de anexar.
- `.prettierignore` exclui `*.md`, então `npm run format` não formata os docs e reescreveria o código da aplicação; evite rodá-lo.

## Setup e verificação

- Node >=20; instale com `npm ci`.
- Banco: `docker compose up -d mysql`, `cp .env.example .env`, depois `npm run db:migrate` (o Prisma carrega o `.env`). O seed é opcional: `npm run db:seed`.
- `npm run dev` e `npm start` passam `--env-file=.env`; `npm test` **não** passa. Exporte o env antes dos testes:
  `set -a; source .env; set +a; npm test`
  Teste focado: `set -a; source .env; set +a; npx vitest run tests/orders.test.ts -t "transitions PENDING"`
- Os testes exigem MySQL e apagam todas as tabelas em `tests/setup.ts` antes de cada teste; nunca aponte para banco compartilhado ou de produção. Eles rodam em fork único, com paralelismo desabilitado.
- Verificação: `npm run lint`; `npm run build` (compila apenas `src`). Não há CI, linter de docs nem script separado de typecheck; valide os docs manualmente contra o checklist original do desafio (atualmente no `README.md` da raiz; preserve-o ou linke para ele antes de substituir o arquivo).
