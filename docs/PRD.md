### PRD: OMS Sistema de Webhooks de Notificação de Pedidos

Versão: 1.0
Data: 23/10/2026
Responsável: Larissa (Tech Lead)

---

### Resumo

O OMS é uma aplicação Node.js e TypeScript, com API REST, Prisma e MySQL, que gerencia pedidos, clientes, produtos e estoque. Hoje os clientes B2B descobrem mudanças de status consultando `GET /orders` periodicamente. Esta feature adiciona webhooks outbound para notificar mudanças de status em tempo quase real, com configuração por endpoint, assinatura HMAC-SHA256, retry com backoff, DLQ e histórico de entregas. A solução usa o MySQL existente e um worker separado, sem introduzir Redis ou outra infraestrutura de fila.

---

### Contexto e problema

Público-alvo
- Clientes B2B que integram seus sistemas ao OMS, incluindo Atlas Comercial, MaxDistribuição e Nova Cargo.
- Usuários autenticados do OMS, incluindo usuários que representam os clientes B2B.
- Times de operação e administração que acompanham falhas e executam replay de DLQ.

Cenários de uso chave
- Cadastrar um endpoint HTTPS e escolher quais status de pedido deseja receber.
- Receber uma notificação quando o status de um pedido muda.
- Validar a assinatura HMAC e deduplicar eventos pelo `X-Event-Id`.
- Consultar as últimas 100 entregas de um endpoint.
- Rotacionar a secret do endpoint com janela de convivência de 24 horas.
- Reprocessar um evento que falhou e foi para a DLQ.

Onde essa feature será implantada
- No OMS existente, aplicação Node.js e TypeScript com Express, Prisma e MySQL. A feature adiciona um módulo de webhooks, tabelas de outbox e DLQ e um processo worker separado, sem criar um novo sistema.

Problema e motivação
- Os clientes B2B fazem polling em `GET /orders` para detectar mudanças de status. Impacto: integração lenta e cara para o cliente. Prioridade: alta.
- A plataforma não notifica mudanças de status de forma externa, então o cliente depende de atualização manual. Impacto: atraso operacional e risco de perda de eventos. Prioridade: alta.
- Não existe histórico de entregas nem mecanismo de replay de falhas. Impacto: diagnóstico e recuperação manuais. Prioridade: média.
- A Atlas sinalizou risco de migração para concorrente se a entrega não ocorrer até o fim do trimestre. Impacto: risco de perda de cliente B2B. Prioridade: alta.

---

### Objetivos e métricas

| Objetivo                                                       | Métrica                                                                       | Meta                                                 |
| -------------------------------------------------------------- | ----------------------------------------------------------------------------- | ---------------------------------------------------- |
| Notificar mudanças de status em tempo quase real               | Latência entre o commit da mudança de status e a entrega bem-sucedida         | Menor que 10 segundos                                |
| Reduzir a dependência de polling                               | Eventos de status assinados por endpoints ativos registrados na outbox        | 100% dos eventos assinados, com tentativa de entrega ou DLQ |
| Recuperar falhas de entrega                                    | Tentativas automáticas antes de mover o evento para a DLQ                     | 1 tentativa inicial + 5 retentativas (6 chamadas) com backoff 1m/5m/30m/2h/12h |
| Dar visibilidade ao cliente                                    | Histórico de entregas por endpoint                                            | Últimas 100 entregas com status, payload, resposta e tempo |
| Proteger a entrega                                             | Requisições entregues com assinatura                                          | 100% com HMAC-SHA256 e secret por endpoint           |

---

### Escopo

Incluso
- CRUD autenticado de configuração de webhook por customer: criar, listar, editar e remover.
- URL somente HTTPS, validada na entrada.
- Lista de status de pedido assinados por endpoint, com filtro na inserção da outbox.
- Secret única por endpoint, gerada pela API e devolvida na criação.
- Rotação de secret com grace period de 24 horas para a secret anterior.
- Inserção de evento na outbox dentro da mesma transação da mudança de status.
- Payload snapshot no momento da inserção, com `event_id` UUID e sem itens do pedido.
- Worker em processo separado, com polling a cada 2 segundos e timeout HTTP de 10 segundos.
- Entrega at-least-once com `X-Event-Id` para deduplicação no cliente.
- Assinatura HMAC-SHA256 sobre o corpo da requisição.
- Headers `X-Signature`, `X-Timestamp`, `X-Webhook-Id`, `X-Event-Id` e `Content-Type: application/json`.
- Retry com 1 tentativa inicial + 5 retentativas (6 chamadas) e backoff 1m/5m/30m/2h/12h.
- DLQ em tabela separada, com payload, motivo da falha e timestamp.
- Replay manual de DLQ restrito a ADMIN e auditado.
- Histórico das últimas 100 entregas por webhook.
- Limite de payload de 64KB, com erro acima do limite.
- Reuso dos padrões existentes: `AppError`, Pino, Zod, error middleware, `requireRole` e estrutura de módulos.
- Observabilidade com logs estruturados, métricas e tracing.

Fora de escopo
- Webhooks inbound, ou seja, clientes enviando eventos para o OMS.
- Envio síncrono do webhook dentro da transação de mudança de status.
- Redis Streams, fila externa ou qualquer nova infraestrutura de mensageria.
- Trigger do banco como mecanismo de notificação do worker.
- Garantia exactly-once.
- Retry com 3 tentativas ou retry infinito.
- Fallback por email para clientes com webhook falhando.
- Rate limiting de saída nesta fase. O tema fica para observação e decisão futura.
- Dashboard visual para o cliente.
- Arquivamento de eventos entregues após 30 dias.
- Escalonamento para múltiplos workers e garantia de ordering global.

---

### Requisitos funcionais

#### FR-001 Cadastrar webhook
Permitir que um usuário autenticado cadastre um endpoint de webhook para um customer, com URL HTTPS e lista de status assinados, recebendo a secret gerada pela API apenas nessa resposta.

**Fluxo principal**
- Receber requisição autenticada com `customerId`, `url` e lista de status.
- Validar a existência do customer.
- Validar que a URL usa HTTPS.
- Validar que os status informados são válidos.
- Gerar uma secret única para o endpoint.
- Persistir a configuração ativa com customer, URL, secret e status assinados.
- Retornar a configuração criada com a secret.

**Fluxos alternativos e exceções**
- Customer inexistente: retornar erro de não encontrado.
- URL com http ou formato inválido: rejeitar com erro de validação.
- Status inválido: rejeitar com erro de validação.
- Requisição sem autenticação: retornar 401.
- Lista de status vazia: rejeitar como configuração sem eventos (hipótese de validação).

**Erros previstos**
- URL não HTTPS.
- Customer não encontrado.
- Status de pedido inválido.
- `customerId` ausente.
- Usuário não autenticado.

**Prioridade:** Alta

---

#### FR-002 Listar webhooks de um customer
Permitir que um usuário autenticado liste os webhooks configurados para um customer.

**Fluxo principal**
- Receber requisição autenticada com `customerId`.
- Buscar os webhooks do customer.
- Retornar a lista com identificador, URL, status assinados e estado ativo.
- Não retornar a secret na listagem (hipótese de segurança).

**Fluxos alternativos e exceções**
- Customer sem webhooks: retornar lista vazia.
- Customer inexistente: retornar erro de não encontrado.
- Requisição sem autenticação: retornar 401.

**Erros previstos**
- Customer não encontrado.
- `customerId` ausente.
- Usuário não autenticado.

**Prioridade:** Média

---

#### FR-003 Editar webhook
Permitir que um usuário autenticado altere URL, lista de status assinados e estado ativo de um webhook.

**Fluxo principal**
- Receber requisição autenticada com o identificador do webhook e os campos a alterar.
- Validar a existência do webhook.
- Validar que a nova URL, se informada, usa HTTPS.
- Validar que os status informados são válidos.
- Persistir as alterações.
- Retornar a configuração atualizada sem a secret.

**Fluxos alternativos e exceções**
- Webhook inexistente: retornar erro de não encontrado.
- URL inválida ou http: rejeitar com erro de validação.
- Status inválido: rejeitar com erro de validação.
- Requisição sem autenticação: retornar 401.

**Erros previstos**
- Webhook não encontrado.
- URL não HTTPS.
- Status de pedido inválido.
- Usuário não autenticado.

**Prioridade:** Média

---

#### FR-004 Remover webhook
Permitir que um usuário autenticado remova um webhook.

**Fluxo principal**
- Receber requisição autenticada com o identificador do webhook.
- Validar a existência do webhook.
- Remover a configuração.
- Retornar confirmação da remoção.

**Fluxos alternativos e exceções**
- Webhook inexistente: retornar erro de não encontrado.
- Requisição sem autenticação: retornar 401.

**Erros previstos**
- Webhook não encontrado.
- Usuário não autenticado.

**Prioridade:** Média

---

#### FR-005 Rotacionar secret do webhook
Permitir que um usuário autenticado gere uma nova secret para um webhook, mantendo a secret anterior válida por 24 horas. A operação será exposta por um endpoint dedicado, com caminho a definir no FDD.

**Fluxo principal**
- Receber requisição autenticada com o identificador do webhook.
- Validar a existência do webhook.
- Gerar uma nova secret única para o endpoint.
- Armazenar a nova secret e manter a anterior válida por 24 horas.
- Retornar a nova secret.

**Fluxos alternativos e exceções**
- Webhook inexistente: retornar erro de não encontrado.
- Requisição sem autenticação: retornar 401.

**Erros previstos**
- Webhook não encontrado.
- Usuário não autenticado.

**Prioridade:** Alta

---

#### FR-006 Registrar evento na outbox na mudança de status
Garantir que toda mudança de status de pedido elegível gere um evento na outbox dentro da mesma transação da mudança de status.

**Fluxo principal**
- O método `changeStatus` inicia a transação no banco.
- A transação valida a transição e atualiza o status do pedido, o histórico e o estoque conforme as regras existentes.
- A transação renderiza um snapshot do payload com `event_id` UUID, `event_type` `order.status_changed`, timestamp ISO 8601, `order_id`, `order_number`, `from_status`, `to_status`, `customer_id` e `total_cents`.
- A transação identifica os webhooks ativos do customer que assinam o novo status.
- Se não houver webhook elegível, nenhum evento é inserido.
- Se houver webhooks elegíveis, a transação insere um evento pendente na `webhook_outbox` para cada webhook que deve receber a notificação.
- A transação é confirmada apenas se a inserção na outbox também for bem-sucedida.

**Fluxos alternativos e exceções**
- Nenhum webhook do customer assina o novo status: não inserir evento.
- Falha na inserção da outbox: reverter toda a transação, incluindo a mudança de status.
- Transição inválida ou estoque insuficiente: manter o comportamento atual da máquina de estados e das validações.
- O payload não inclui itens do pedido. Detalhes podem ser consultados depois em `GET /orders/:id`.

**Erros previstos**
- Transição de status inválida.
- Estoque insuficiente.
- Falha ao inserir na outbox.
- Payload do evento inválido ou incompleto.

**Prioridade:** Alta

---

#### FR-007 Entregar webhook pelo worker
Processar eventos pendentes da outbox em um worker separado e enviar requisições HTTP assinadas para os endpoints configurados.

**Fluxo principal**
- O worker roda em processo separado da API e consulta a outbox a cada 2 segundos.
- O worker busca os eventos pendentes mais antigos em lotes pequenos.
- Para cada evento, o worker carrega a configuração do webhook e a secret vigente.
- O worker serializa o payload em JSON e calcula o HMAC-SHA256 sobre o corpo com a secret do endpoint.
- O worker envia POST para a URL do webhook com `Content-Type: application/json` e os headers `X-Signature`, `X-Timestamp`, `X-Webhook-Id` e `X-Event-Id`.
- O worker aguarda no máximo 10 segundos pela resposta.
- Resposta 2xx: marcar o evento como entregue.
- Erro de rede, timeout ou resposta 5xx: registrar a falha e encaminhar o evento para a política de retry. Resposta 4xx: registrar a falha; a política de retry para 4xx está pendente e deve ser definida no FDD.

**Fluxos alternativos e exceções**
- Timeout de 10 segundos: tratar como falha e aplicar retry.
- Resposta 5xx: tratar como falha e aplicar retry.
- Resposta 4xx: tratar como falha; a política de retry está pendente.
- Erro de rede: tratar como falha e aplicar retry.
- Payload acima de 64KB: não truncar e registrar erro (hipótese: falha permanente, sem retry).

**Erros previstos**
- Timeout de 10 segundos.
- Resposta HTTP 5xx.
- Resposta HTTP 4xx com política de retry pendente.
- Erro de rede.
- Payload acima de 64KB.

**Prioridade:** Alta

---

#### FR-008 Aplicar retry com backoff e mover para DLQ
Reenviar entregas com falhas retryáveis (5xx, timeout e erro de rede) com 1 tentativa inicial + 5 retentativas (6 chamadas) e backoff progressivo, persistindo na DLQ os eventos que esgotarem as tentativas. A política de retry para 4xx está pendente no FDD.

**Fluxo principal**
- Em caso de falha retryável, registrar a tentativa e o motivo.
- Se ainda houver retentativas disponíveis, reagendar o evento com os intervalos 1 minuto, 5 minutos, 30 minutos, 2 horas e 12 horas.
- Se a entrega tiver sucesso antes do limite, marcar como entregue e não enviar para a DLQ.
- Se as 5 retentativas falharem após a tentativa inicial, mover o evento para a tabela `webhook_dead_letter` com payload, motivo da falha e timestamp.
- Remover o evento da fila ativa da outbox após a movimentação para a DLQ.

**Fluxos alternativos e exceções**
- Sucesso em uma tentativa intermediária: encerrar o ciclo de retry.
- Cliente continua indisponível após as cinco retentativas: manter o evento na DLQ até replay manual.
- Falha ao gravar na DLQ: não perder o evento, registrar erro e manter o evento para nova tentativa (hipótese operacional).

**Erros previstos**
- Esgotamento das 5 retentativas.
- Falha persistente do cliente.
- Falha de persistência na DLQ.

**Prioridade:** Alta

---

#### FR-009 Consultar histórico de entregas
Permitir que um usuário autenticado consulte as últimas 100 entregas de um webhook.

**Fluxo principal**
- Receber requisição autenticada com o identificador do webhook.
- Validar a existência do webhook.
- Buscar as últimas 100 entregas, da mais recente para a mais antiga.
- Retornar sucesso ou falha, payload, resposta do cliente e tempo de resposta.

**Fluxos alternativos e exceções**
- Webhook sem entregas: retornar lista vazia.
- Webhook inexistente: retornar erro de não encontrado.
- Requisição sem autenticação: retornar 401.

**Erros previstos**
- Webhook não encontrado.
- Usuário não autenticado.

**Prioridade:** Média

---

#### FR-010 Reprocessar evento da DLQ
Permitir que um usuário com role ADMIN recoloque um evento da DLQ na outbox como pendente, com registro de auditoria.

**Fluxo principal**
- Receber requisição autenticada com o identificador do evento na DLQ.
- Validar que o usuário tem role ADMIN.
- Validar a existência do evento na DLQ.
- Recolocar o evento na outbox com status pendente.
- Registrar em log quem executou o replay, quando e qual evento foi reprocessado.
- Retornar confirmação do replay.

**Fluxos alternativos e exceções**
- Usuário sem role ADMIN: retornar 403.
- Evento inexistente na DLQ: retornar erro de não encontrado.
- Requisição sem autenticação: retornar 401.

**Erros previstos**
- Acesso negado para usuário sem role ADMIN.
- Evento de DLQ não encontrado.
- Usuário não autenticado.

**Prioridade:** Alta

---

### Requisitos não funcionais

Performance
- API de configuração: p95 menor que 150 ms (hipótese, pois não foi definido na reunião).
- Entrega de webhook: latência entre o commit da mudança de status e a resposta 2xx menor que 10 segundos, conforme o requisito de tempo real citado na reunião.
- Worker: polling a cada 2 segundos, leitura em lotes pequenos e timeout HTTP de 10 segundos.
- Payload: máximo de 64KB por evento, sem truncamento. Eventos acima do limite devem gerar erro.

Disponibilidade
- 99.9% mensal para a API de configuração e para o processamento de eventos (hipótese, pois não foi definido na reunião).
- Eventos ficam persistidos na outbox durante indisponibilidade do worker e são processados quando ele voltar.
- O worker é um ponto único de processamento nesta fase. A outbox evita perda de eventos, mas não garante entrega durante a indisponibilidade do worker.

Segurança e autorização
- Autenticação JWT Bearer obrigatória em todos os endpoints da feature, usando o middleware `authenticate` existente.
- Replay de DLQ exige role ADMIN via `requireRole`.
- CRUD de configuração e consulta de entregas permitem qualquer role autenticada nesta fase.
- HMAC-SHA256 sobre o corpo JSON, com secret única por endpoint.
- Secret gerada pela API e devolvida apenas na criação e na rotação.
- URL somente HTTPS. HTTP deve ser rejeitado na validação.
- Rotação de secret com grace period de 24 horas.
- Auditoria do replay de DLQ com usuário, timestamp e identificador do evento.
- Segredos não devem aparecer em logs (hipótese de implementação).

Observabilidade
- Logs estruturados com Pino, incluindo `event_id`, `webhook_id`, `order_id`, tentativa, status HTTP e duração.
- Métricas: eventos pendentes na outbox, sucesso e falha de entrega, tentativas de retry, tamanho da DLQ, latência de entrega e tempo de resposta do cliente.
- Tracing distribuído ponta a ponta para correlacionar mudança de status, evento na outbox e entrega HTTP (hipótese, pois o código atual tem Pino e `X-Request-Id`, mas não tracing).

Confiabilidade e integridade de dados
- Inserção na outbox na mesma transação da mudança de status, com rollback em caso de falha.
- Garantia at-least-once. Duplicatas são possíveis e devem ser deduplicadas pelo cliente usando `X-Event-Id`.
- Payload snapshot no momento da inserção. Se o pedido mudar depois, o evento mantém o estado da época da mudança de status.
- Ordenação por `order_id` enquanto houver um único worker. Não há garantia de ordering global.
- Retry com 1 tentativa inicial + 5 retentativas (6 chamadas) e DLQ para falhas permanentes.
- Eventos entregues não são arquivados nesta fase. O arquivamento após 30 dias fica fora de escopo.

Compatibilidade e portabilidade
- API REST JSON sob o prefixo existente `/api/v1`.
- Payload com `event_type` `order.status_changed` e timestamp ISO 8601.
- Stack existente Node.js, TypeScript, Express, Prisma e MySQL 8, sem Redis ou fila externa.
- Cliente deve expor endpoint HTTPS e suportar HMAC-SHA256 e deduplicação por `X-Event-Id`.
- O worker usa a mesma `DATABASE_URL`, com um PrismaClient separado por processo.

Compliance
- Trilha de auditoria do replay de DLQ com quem executou, quando e qual evento.
- O payload expõe dados básicos do pedido e `customer_id`. O acesso à configuração e ao histórico deve seguir a autenticação e autorização do OMS.
- Não foram definidos requisitos regulatórios específicos adicionais na reunião.

Acessibilidade no frontend consumidor
- Não se aplica nesta fase, pois a entrega é uma API e o dashboard visual está fora de escopo. A documentação no portal deve incluir exemplos de payload, headers e validação de assinatura.

---

### Arquitetura e abordagem

Abordagem
- Outbox no MySQL existente, transação atômica com `changeStatus`, worker separado em polling a cada 2 segundos e entrega HTTP assíncrona. Monólito existente mais processo worker, sem Redis, fila externa ou trigger de banco.

Componentes
- API Express existente, com novo módulo `src/modules/webhooks` seguindo o padrão controller, service, repository, routes e schemas.
- Tabela `webhook_outbox` no MySQL, com IDs UUID, status pendente, processando, falhou e entregue, além de índices em status e `created_at`.
- Tabela `webhook_dead_letter` no MySQL, com payload, motivo da falha e timestamp.
- Tabela de configuração de webhooks com customer, URL, secret, status assinados e estado ativo.
- Registro de entregas para alimentar o histórico das últimas 100 tentativas.
- Função `publishWebhookEvent(tx, order, fromStatus, toStatus)` chamada dentro da transação de `changeStatus`, recebendo o tx client.
- Worker `src/worker.ts`, com lógica de processamento em `src/modules/webhooks` e PrismaClient próprio.
- Reuso de `AppError`, Pino, error middleware, Zod, `requireRole` e códigos de erro `WEBHOOK_*`.

Integrações
- `src/modules/orders/order.service.ts` no método `changeStatus`, para inserir a outbox na mesma transação.
- MySQL via Prisma, compartilhado entre API e worker.
- Endpoint HTTPS do cliente, via POST JSON assinado.
- `src/middlewares/auth.middleware.ts` para `authenticate` e `requireRole`.
- `src/middlewares/error.middleware.ts` e `src/shared/logger/index.ts` para erros e logs.
- `src/routes/index.ts` e `src/app.ts` para montar as rotas sob `/api/v1`.

### Decisões e trade-offs

#### Decisão: Usar outbox no MySQL em vez de envio síncrono ou fila externa
- **Justificativa:** A outbox é gravada na mesma transação da mudança de status, então o evento existe se e somente se a mudança for confirmada. O MySQL já está em produção e a equipe é pequena, então Redis ou outra fila seria infraestrutura adicional sem necessidade.
- **Trade-off:** A entrega passa a depender de polling, com latência mínima de 2 segundos, e a tabela precisa de índices, monitoramento e arquivamento futuro. O envio não é imediato como em uma chamada síncrona.

#### Decisão: Worker separado em polling de 2 segundos
- **Justificativa:** O MySQL não tem listener nativo para notificar um processo externo, e um trigger de banco não resolve esse problema. O polling de 2 segundos atende ao requisito de tempo real abaixo de 10 segundos e o worker separado continua funcionando quando a API reinicia.
- **Trade-off:** Há latência mínima de 2 segundos e carga periódica de consulta no banco. O worker é um processo adicional para operar e monitorar.

#### Decisão: Retry com 1 tentativa inicial + 5 retentativas, backoff 1m/5m/30m/2h/12h e DLQ separada
- **Justificativa:** A janela de aproximadamente 15 horas cobre indisponibilidades planejadas e evita que eventos fiquem pendurados para sempre. A DLQ separada mantém a outbox principal limpa e preserva payload, motivo e timestamp para diagnóstico e replay.
- **Trade-off:** Um cliente que fique fora por mais de aproximadamente 15 horas depende de replay manual. A operação passa a ter mais uma tabela e um fluxo administrativo para monitorar.

#### Decisão: HMAC-SHA256 com secret única por endpoint e rotação com grace period de 24 horas
- **Justificativa:** O cliente consegue validar que a requisição veio do OMS e que o payload não foi adulterado. A secret por endpoint limita o impacto de um vazamento a um único cadastro.
- **Trade-off:** O cliente precisa armazenar a secret com segurança e implementar a verificação HMAC. A rotação exige uma janela de convivência de 24 horas, na qual duas secrets são válidas.

#### Decisão: Garantia at-least-once com deduplicação por X-Event-Id
- **Justificativa:** Exactly-once exigiria coordenação entre OMS e cliente e seria mais complexo. At-least-once com um identificador único por evento é o padrão de mercado e cobre a maior parte dos casos.
- **Trade-off:** O cliente pode receber o mesmo evento mais de uma vez e precisa deduplicar pelo `X-Event-Id`. A responsabilidade de idempotência fica parcialmente no cliente.

#### Decisão: Payload snapshot no momento da inserção e filtro de status na outbox
- **Justificativa:** O snapshot preserva o estado do pedido no momento da mudança de status, mesmo que o pedido mude depois. O filtro na inserção evita armazenar eventos que nenhum webhook do customer assina.
- **Trade-off:** O payload pode ficar desatualizado em relação ao estado atual do pedido, e o cliente precisa consultar `GET /orders/:id` para detalhes. Eventos de status não assinados não ficam disponíveis para envio posterior sem uma nova mudança de status.

#### Decisão: Reutilizar os padrões existentes do projeto
- **Justificativa:** O OMS já tem `AppError`, error middleware, Pino, Zod, `requireRole` e um padrão de módulos bem definido. Reutilizar esses componentes mantém consistência, reduz código novo e faz o tratamento de erros funcionar sem mudanças no middleware central.
- **Trade-off:** O módulo de webhooks fica acoplado ao padrão do monólito e ao mesmo ciclo de deploy da API, em vez de evoluir como um serviço independente.

---

### Dependências

#### Externa: Endpoints HTTPS dos clientes B2B
Os clientes precisam expor endpoints HTTPS disponíveis para receber as notificações e implementar a validação da assinatura HMAC e a deduplicação por `X-Event-Id`. Sem isso, as entregas falham e os eventos acabam na DLQ.

#### Organizacional: Revisão de segurança da Sofia
A Sofia precisa revisar o código de segurança, incluindo HMAC e geração de secret, por pelo menos dois dias úteis antes do deploy. Essa revisão é uma condição para liberar a feature em produção.

#### Organizacional: Documentação no portal do desenvolvedor
O Marcos precisa documentar a integração no portal do desenvolvedor, incluindo payload, headers, validação de assinatura, deduplicação e comportamento de retry. Os clientes dependem dessa documentação para integrar corretamente.

#### Técnica: Banco MySQL existente e Prisma
A feature depende do MySQL já usado pelo OMS e do Prisma para acesso a dados. O worker usa a mesma `DATABASE_URL`, mas abre um PrismaClient separado porque roda em outro processo Node.js.

#### Técnica: Componentes e padrões existentes do OMS
A feature depende de `AppError`, Pino, error middleware, Zod, `requireRole`, do padrão de módulos e da máquina de estados de pedidos. Esses componentes precisam continuar disponíveis e compatíveis com o novo módulo.

#### Técnica: Novas tabelas e migração Prisma
As tabelas `webhook_outbox`, `webhook_dead_letter`, a configuração de webhooks e o registro de entregas precisam ser modelados e migrados antes da entrada do worker em produção.

---

### Riscos e mitigação

Probabilidades e impactos são estimativas iniciais (hipótese), pois não foram quantificados na reunião.

#### Cliente indisponível ou lento durante a entrega
- **Probabilidade:** Alta
- **Impacto:** Eventos ficam pendentes, a entrega atrasa e, após as tentativas, o cliente perde a notificação até um replay manual.
- **Mitigação:**
  - Aplicar 1 tentativa inicial + 5 retentativas com backoff de 1m/5m/30m/2h/12h.
  - Usar timeout HTTP de 10 segundos para não travar o worker.
  - Persistir falhas permanentes na DLQ para replay posterior.
- **Plano de contingência:** O time de operação identifica os eventos na DLQ e um ADMIN executa o replay depois que o cliente normalizar o endpoint.

#### Vazamento de secret ou comprometimento do endpoint do cliente
- **Probabilidade:** Média
- **Impacto:** Terceiros podem forjar requisições ou acessar dados de pedidos expostos ao endpoint comprometido.
- **Mitigação:**
  - Usar uma secret única por endpoint.
  - Assinar o corpo com HMAC-SHA256.
  - Exigir HTTPS em todas as URLs.
  - Permitir rotação de secret com grace period de 24 horas.
  - Manter o payload enxuto, sem itens do pedido.
- **Plano de contingência:** O cliente solicita a rotação da secret e a antiga deixa de ser válida após 24 horas. Se necessário, o webhook pode ser desativado e recriado.

#### Divergência entre a mudança de status e o registro na outbox
- **Probabilidade:** Baixa
- **Impacto:** O cliente não recebe uma notificação que deveria ter sido enviada, gerando inconsistência entre o OMS e o sistema do cliente.
- **Mitigação:**
  - Inserir a outbox na mesma transação SQL do `changeStatus`.
  - Reverter a transação inteira se a inserção na outbox falhar.
  - Cobrir o cenário de rollback com testes de integração.
- **Plano de contingência:** Investigar a divergência comparando `order_status_history` com a `webhook_outbox` e tratar o evento faltante manualmente com o time de plataforma.

#### Acúmulo de eventos na outbox e latência acima da meta
- **Probabilidade:** Média
- **Impacto:** Entregas atrasam, a latência ultrapassa 10 segundos e o cliente pode receber uma sequência de eventos fora do ritmo esperado.
- **Mitigação:**
  - Usar índices em status e `created_at`.
  - Ler somente eventos pendentes e em lotes pequenos.
  - Monitorar tamanho da outbox, latência de entrega e volume por cliente.
  - Manter um único worker nesta fase para preservar a ordering por `order_id`.
- **Plano de contingência:** Priorizar o processamento dos eventos mais antigos, avaliar aumento de capacidade e, em fase futura, particionar por `order_id` ou usar lock pessimista.

#### Duplicidade de entrega não tratada pelo cliente
- **Probabilidade:** Média
- **Impacto:** O cliente processa o mesmo evento mais de uma vez e pode gerar efeitos duplicados no sistema dele.
- **Mitigação:**
  - Enviar `X-Event-Id` com UUID único por evento.
  - Documentar a garantia at-least-once e a deduplicação no portal do desenvolvedor.
  - Manter o payload snapshot para que a duplicata represente o mesmo estado.
- **Plano de contingência:** O suporte orienta o cliente a deduplicar pelo `X-Event-Id` e a revisar o processamento idempotente antes de solicitar replay.

#### Perda de clientes B2B por atraso na entrega
- **Probabilidade:** Média
- **Impacto:** A Atlas e outros clientes B2B podem migrar para um concorrente, afetando a receita e a reputação do OMS.
- **Mitigação:**
  - Entregar em três sprints, incluindo a revisão de segurança da Sofia.
  - Priorizar o escopo mínimo: outbox, worker, CRUD, HMAC, retry e DLQ.
  - Comunicar o progresso aos clientes por meio do Marcos.
- **Plano de contingência:** Renegociar prazo e escopo com os clientes, priorizando os itens de maior valor e mantendo a comunicação transparente sobre o andamento.

---

### Critérios de aceitação
Checklist objetivo que define se a feature está pronta.

- O CRUD de webhooks está disponível e exige autenticação JWT.
- A criação de webhook retorna a secret gerada apenas na resposta de criação.
- A URL de webhook é validada como HTTPS e URLs http são rejeitadas.
- Cada endpoint tem uma secret única e a rotação mantém a secret anterior válida por 24 horas.
- A mudança de status insere o evento na outbox dentro da mesma transação do `changeStatus`.
- Se nenhum webhook do customer assina o novo status, nenhum evento é inserido na outbox.
- Se a inserção na outbox falhar, a mudança de status é revertida.
- O worker roda em processo separado e consulta a outbox a cada 2 segundos.
- A entrega usa POST JSON com `X-Signature`, `X-Timestamp`, `X-Webhook-Id`, `X-Event-Id` e `Content-Type: application/json`.
- A assinatura HMAC-SHA256 é calculada sobre o corpo e pode ser validada pelo cliente.
- O timeout de cada chamada HTTP é de 10 segundos.
- Falhas retryáveis (5xx, timeout e erro de rede) aplicam 1 tentativa inicial + 5 retentativas (6 chamadas) com os intervalos 1m/5m/30m/2h/12h. A política de retry para 4xx está pendente.
- Após as 5 retentativas sem sucesso de uma falha retryável, o evento é persistido na `webhook_dead_letter` com payload, motivo e timestamp.
- Payloads acima de 64KB são rejeitados sem truncamento.
- O endpoint de replay de DLQ exige role ADMIN e registra em log quem executou o replay.
- O histórico retorna as últimas 100 entregas de um webhook com status, payload, resposta e tempo de resposta.
- O comportamento de ordering por `order_id` e a limitação de single-worker estão documentados.
- Os testes unitários, de integração e de segurança definidos na estratégia passam no ambiente de teste.

---

### Testes e validação

Tipos de teste obrigatórios
- Testes unitários para a máquina de estados, filtro de status, montagem do payload, cálculo de backoff, geração e validação de HMAC e geração de secret.
- Testes de integração para CRUD de webhooks, autenticação, transação de `changeStatus` com outbox, worker com servidor HTTP de teste, retry, DLQ, histórico de entregas e replay.
- Testes de segurança para autenticação, autorização ADMIN no replay, rejeição de URL http, validação de assinatura, rotação de secret e não exposição de secret em respostas de listagem.
- Testes de contrato para payload, headers, status codes e formato dos eventos enviados ao cliente.
- Teste de carga do worker e da outbox para validar latência, backlog e comportamento sob volume representativo de eventos.

Estratégia de validação
- TDD para as regras críticas de transição de status, inserção atômica na outbox, filtro de eventos, assinatura e retry.
- Integração com MySQL local, usando as migrações Prisma e um servidor HTTP de teste no lugar do endpoint do cliente. Nunca usar banco de produção nos testes.
- QA manual guiado por roteiro, cobrindo cadastro, edição, entrega, falha, retry, DLQ, replay e rotação de secret.
- Revisão de segurança da Sofia antes do deploy, com pelo menos dois dias úteis reservados para HMAC e geração de secret.
- Validação exploratória interna com um cliente simulado que verifica assinatura, deduplicação e resposta 2xx.
