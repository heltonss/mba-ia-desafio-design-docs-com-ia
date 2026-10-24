# RFC — Sistema de Webhooks de Notificação de Pedidos

## Metadados

| Campo | Valor |
| --- | --- |
| Título | Webhooks outbound de notificação de mudança de status de pedidos |
| Autor | Larissa (Tech Lead) |
| Status | Em revisão |
| Data | 24/10/2026 |
| Revisores | Marcos (Product Manager), Bruno (Engenheiro Pleno, time de Pedidos), Diego (Engenheiro Sênior, time de Plataforma), Sofia (Engenheira de Segurança) |
| Escopo do documento | Proposta de arquitetura (o quê propomos e por quê). O detalhamento de implementação fica no FDD. |

---

## Resumo executivo (TL;DR)

Três clientes B2B (Atlas Comercial, MaxDistribuição e Nova Cargo) querem ser notificados quando o status de um pedido muda, em vez de ficar fazendo polling em `GET /orders`. Propomos implementar **webhooks outbound** sobre um **padrão outbox transacional no MySQL existente**: a mudança de status grava o evento na mesma transação SQL; um **worker em processo separado** faz polling da outbox a cada 2 segundos e entrega a notificação via HTTPS com assinatura **HMAC-SHA256**.

A entrega é **at-least-once** (com `X-Event-Id` para deduplicação no cliente) e usa **retry com backoff** (`1m/5m/30m/2h/12h`) seguido de **DLQ em tabela separada** com replay manual restrito a ADMIN. A solução **reutiliza** a stack e os padrões existentes (AppError, Pino, Zod, error middleware, `requireRole`, estrutura de módulos) e **não introduz** nova infraestrutura de mensageria (sem Redis, sem fila externa, sem trigger de banco). A entrega estimada é de **três sprints**, incluindo revisão de segurança.

---

## Contexto e problema

Os clientes B2B hoje descobrem mudanças de status consultando `GET /orders` periodicamente. Isso torna a integração lenta e cara para eles e não garante que toda mudança seja observada. A Atlas Comercial sinalizou que pode migrar para um concorrente se a notificação em tempo real não for entregue até o fim do trimestre.

O requisito de negócio é "tempo real" com tolerância de até 10 segundos. A feature é somente **outbound** (o OMS envia; os clientes não enviam para nós). O OMS atual não possui nenhum mecanismo de eventos, fila, webhook ou histórico de entregas — essa lacuna é exatamente o que esta feature preenche.

O problema técnico central é **como notificar sem acoplar a latência e a disponibilidade do cliente à operação crítica de mudança de status**, que hoje já é pesada (atualiza `orders`, insere em `order_status_history` e altera o estoque) e não pode sofrer rollback por causa de um cliente fora do ar.

---

## Proposta técnica

A solução adiciona quatro peças ao OMS existente, sem criar um novo sistema:

1. **Outbox transacional no MySQL.** Dentro da transação de `changeStatus`, além de atualizar o pedido e o histórico, gravamos uma linha de evento na tabela `webhook_outbox`. Se a transação confirmar, o evento existe; se reverter, o evento desaparece junto. Não há divergência possível entre status mudado e evento registrado. O payload é um **snapshot renderizado na inserção** e o evento é filtrado pelos status que cada endpoint assina — se nenhum webhook do customer assina o novo status, nada é inserido.

2. **Worker em processo separado.** Um processo Node independente (`src/worker.ts`, script `npm run worker`) faz **polling da outbox a cada 2 segundos**, lê os eventos pendentes mais antigos em lote pequeno e dispara as chamadas HTTP. Rodar separado da API garante que o reinício da API não derruba o worker. Por enquanto é **single-worker**, o que dá ordenação implícita por `order_id` (não há garantia de ordenação global).

3. **Entrega com retry e DLQ.** Cada entrega tem timeout HTTP de 10 segundos. Em caso de falha retryável (5xx, timeout, erro de rede), aplicamos **1 tentativa inicial + 5 retentativas** com backoff `1m/5m/30m/2h/12h` (~15h de janela). Esgotadas as tentativas, o evento vai para `webhook_dead_letter` (tabela separada com payload, motivo e timestamp), com **replay manual** via endpoint ADMIN auditado.

4. **Segurança na borda.** A URL do endpoint é obrigatoriamente HTTPS. Cada endpoint tem **secret única** gerada pela API, e cada entrega é assinada com **HMAC-SHA256** sobre o corpo (`X-Signature`), além de `X-Timestamp`, `X-Webhook-Id` e `X-Event-Id`. A secret é rotacionável com **grace period de 24 horas**. O `X-Event-Id` (UUID) permite deduplicação no cliente, fechando a garantia at-least-once.

O detalhamento de como construir (contratos HTTP, matriz de erros `WEBHOOK_*`, fluxos, integração com o código) está no **FDD**. Este RFC fica na decisão arquitetural.

---

## Alternativas consideradas

| Alternativa | Trade-off que levou ao descarte |
| --- | --- |
| **Envio síncrono dentro de `changeStatus`** | Acopla latência e disponibilidade do cliente à operação de mudança de status; um cliente lento travaria mudanças de outros pedidos e, se o cliente estivesse fora do ar, não haveria como reverter a mudança de status apenas por causa da notificação. |
| **Redis Streams / fila externa (Redis Cluster)** | Exige subir e operar nova infraestrutura de mensageria. Para um time pequeno, é overengineering quando o MySQL existente resolve com outbox. |
| **Trigger do banco como notificador** | MySQL não tem listener nativo tipo `NOTIFY/LISTEN` do Postgres; um trigger só executa SQL e não notifica processo externo. Seria necessário improvisar (arquivo/endpoint) e fica frágil. O polling de 2s atende o requisito de 10s. |
| **Retry com 3 tentativas** | Insuficiente para cobrir indisponibilidades reais (já houve manutenção planejada de 2h em cliente). |
| **Retry infinito com backoff** | Evento de cliente que "sumiu" fica pendurado para sempre. |
| **Entrega exactly-once** | Exigiria coordenação dos dois lados (OMS + cliente) e aumenta muito a complexidade. At-least-once com `X-Event-Id` é o padrão de mercado e resolve a grande maioria dos casos. |

---

## Questões em aberto

Pontos levantados na reunião e não decididos (ou explicitamente adiados), que precisam ser fechados antes/durante a implementação:

1. **Rate limiting de saída.** Um cliente com muitos pedidos em pouco tempo pode receber uma rajada de chamadas. Ficou definido **observar e decidir depois**; não entra nesta fase.
2. **Fallback por email para webhook com falha.** Adiado para uma próxima fase, após medir o impacto.
3. **Formato de convivência das duas secrets durante a rotação (24h).** Como o `X-Signature` comporta a secret anterior e a nova em paralelo ainda não foi definido.
4. **Política de retry para resposta 4xx do cliente.** A reunião detalhou 5xx/timeout/erro de rede, mas não fechou o comportamento para 4xx.
5. **Recuperação de eventos presos em `PROCESSANDO`** se o worker cair no meio de uma entrega (mecanismo de lease/requeue não definido).
6. **Ordenação por `order_id` vs. retry.** Um retry atrasado pode deixar um evento posterior ultrapassar o anterior; a contradição não foi resolvida.

---

## Impacto e riscos

**Impacto operacional**

- Novo processo (worker) a operar e monitorar, além da API.
- Novas tabelas no MySQL: configuração de webhooks, `webhook_outbox`, histórico de entregas e `webhook_dead_letter`.
- Latência mínima de entrega passa a ser ~2s (polling), aceitável diante da meta de 10s.

**Riscos**

| Risco | Probabilidade | Impacto | Mitigação |
| --- | --- | --- | --- |
| Worker único é ponto único de processamento (backlog se parar) | Média | Alto | Processo separado com supervisão; outbox persiste eventos até o worker voltar; monitorar backlog. |
| Acoplar a inserção na outbox ao `changeStatus` pode bloquear a operação principal | Baixa | Alto | Inserção simples e atômica na transação; testes de rollback; monitorar falhas de escrita na outbox. |
| Vazamento de secret de um endpoint | Média | Alto | Secret única por endpoint, HMAC-SHA256, HTTPS obrigatório, rotação com grace period e payload enxuto (sem itens). |
| Duplicidade de entrega não tratada pelo cliente | Média | Médio | `X-Event-Id` estável entre retries/replay; documentar deduplicação no portal. |
| Cliente indisponível por mais de ~15h | Média | Médio | Evento vai para DLQ e é recuperado por replay manual ADMIN. |

---

## Decisões relacionadas

Cada decisão fechada abaixo tem um ADR dedicado:

- [ADR-001 — Outbox transacional no MySQL](adrs/ADR-001-outbox-transacional-no-mysql.md)
- [ADR-002 — Worker em processo separado com polling](adrs/ADR-002-worker-em-processo-separado-com-polling.md)
- [ADR-003 — Retry com backoff e DLQ](adrs/ADR-003-retry-com-backoff-e-dlq.md)
- [ADR-004 — HMAC-SHA256 com secret por endpoint](adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md)
- [ADR-005 — Entrega at-least-once com X-Event-Id](adrs/ADR-005-entrega-at-least-once-com-x-event-id.md)
- [ADR-006 — Reuso dos padrões existentes do projeto](adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md)

---

*Solicito revisão e comentários dos revisores listados acima, em especial da Sofia quanto aos itens de segurança antes de seguir para o FDD.*
