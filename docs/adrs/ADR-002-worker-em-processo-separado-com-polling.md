# ADR-002 — Worker em processo separado com polling

- **Status:** Aceita
- **Data:** 24/10/2026
- **Decisores:** Larissa, Diego, Bruno ([09:09]–[09:13])

## Contexto

A outbox registra os eventos na mesma transação da mudança de status, mas alguém precisa lê-los e disparar as chamadas HTTP para os clientes. É preciso decidir como esse processamento acontece e onde ele roda. O requisito de latência é "tempo real" com tolerância de até 10 segundos ([09:02] Marcos).

## Decisão

Criar um **worker em processo separado da API**, que faz **polling da outbox a cada 2 segundos**, buscando os eventos pendentes mais antigos em lote pequeno, processando e marcando como entregue ([09:09] Diego). O worker roda como uma entrypoint nova — análoga a `src/server.ts` — em `src/worker.ts`, com script `npm run worker`, conectando no mesmo banco e na mesma stack, mas em processo Node próprio com PrismaClient separado ([09:11] Larissa, [09:11] Diego, [09:30] Bruno).

Nesta fase o worker é **único (single-worker)**. Com um único worker processando em ordem de `created_at`, há **ordenação implícita por `order_id`**, mas **não há garantia de ordenação global** ([09:12] Diego, [09:13] Larissa). A evolução para múltiplos workers (particionamento por `order_id` ou lock pessimista) fica para o futuro.

## Alternativas Consideradas

- **Rodar o worker dentro da mesma instância da API:** descartado porque o reinício da API derrubaria o processamento junto ([09:11] Diego).
- **Trigger do banco como notificador:** descartado porque MySQL não tem listener nativo; um trigger só executa SQL e não avisa processo externo. O polling de 2s atende o requisito de 10s ([09:09] Diego).
- **Reatividade via mensageria (push):** implicaria a infraestrutura de fila já descartada no ADR-001.

## Consequências

**Positivas**

- Isolamento do processamento: reinício da API não interrompe o worker, e reinício do worker não perde eventos persistidos na outbox.
- Simples de operar: um único processo, sem coordenação distribuída.
- Ordenação por `order_id` garantida enquanto houver um único worker.

**Negativas**

- Latência mínima de 2s entre a mudança de status e a tentativa de entrega.
- Carga periódica de consulta no banco (mitigada por índices em `status` e `created_at`).
- Worker único é um ponto único de processamento: se parar, o backlog cresce até voltar.
- Não há garantia de ordenação global nem escalabilidade horizontal sem redesenho futuro.
