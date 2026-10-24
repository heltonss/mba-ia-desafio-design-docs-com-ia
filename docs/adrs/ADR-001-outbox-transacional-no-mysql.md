# ADR-001 — Outbox transacional no MySQL

- **Status:** Aceita
- **Data:** 24/10/2026
- **Decisores:** Larissa, Diego, Bruno ([09:06]–[09:08])

## Contexto

Os clientes B2B hoje descobrem mudanças de status fazendo polling em `GET /orders`, o que é lento e caro. A notificação precisa ser "tempo real" com tolerância de até 10 segundos ([09:02] Marcos).

O método `changeStatus` em `src/modules/orders/order.service.ts` já executa uma transação pesada: atualiza `orders`, insere em `order_status_history` e altera `stock_quantity` dos produtos. Acrescentar uma chamada HTTP nessa transação acoplaria a latência e a disponibilidade do cliente à operação crítica, e não haveria como reverter a mudança de status só porque o cliente está fora do ar ([09:04] Bruno, [09:06] Diego).

## Decisão

Adotar o padrão **outbox transacional no MySQL existente**. Quando o status de um pedido muda, dentro da mesma transação SQL que atualiza `orders` e `order_status_history`, gravamos uma linha de evento na tabela `webhook_outbox`. Um worker separado lê essa tabela e dispara as chamadas HTTP. Se a transação confirmar, o evento foi registrado; se reverter, o evento desaparece junto — não há inconsistência possível ([09:06] Diego).

Detalhes complementares fechados na reunião:

- A tabela tem índices em `status` (pendente/processando/falhou/entregue) e em `created_at` ([09:08] Diego).
- O identificador da outbox é **UUID**, seguindo o padrão do projeto ([09:51] Larissa).
- O payload é um **snapshot renderizado na inserção**, não resolvido no momento do envio ([09:52] Larissa, [09:52] Diego).
- O filtro de status assinados por cada webhook é aplicado na **inserção**, para não gravar eventos que nenhum endpoint assina ([09:34] Bruno, [09:34] Diego).

## Alternativas Consideradas

- **Envio síncrono dentro de `changeStatus`:** descartado porque um cliente lento travava mudanças de status de outros pedidos e, com cliente fora do ar, não havia como reverter a mudança de status sem perder a notificação ([09:04] Bruno, [09:06] Diego).
- **Redis Streams / fila externa (Redis Cluster):** descartado por exigir infraestrutura adicional de mensageria, o que é overengineering para um time pequeno quando o MySQL já resolve ([09:07] Larissa, [09:07] Diego).
- **Trigger do banco como notificador:** descartado porque MySQL não tem listener nativo (tipo `NOTIFY/LISTEN`); o trigger só executa SQL e não notifica um processo externo ([09:09] Diego).

## Consequências

**Positivas**

- Atomicidade forte entre a mudança de status e o registro do evento: status confirmado tem evento, rollback remove o evento.
- Nenhuma infraestrutura nova de mensageria; reutiliza o MySQL e o Prisma já em produção.
- Desacopla a entrega da operação síncrona da API.

**Negativas**

- A entrega passa a depender de polling, com latência mínima de ~2s (aceitável diante da meta de 10s).
- A tabela cresce com o tempo e exigirá arquivamento futuro (fora de escopo desta feature) ([09:08] Diego).
- A inserção na outbox passa a fazer parte do caminho crítico do `changeStatus`; uma falha nela reverte a operação principal.

**Referências de código:** `src/modules/orders/order.service.ts` (transação de `changeStatus`), `src/modules/orders/order.status.ts` (máquina de estados).
