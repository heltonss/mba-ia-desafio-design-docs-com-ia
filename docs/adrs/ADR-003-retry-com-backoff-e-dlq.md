# ADR-003 — Retry com backoff e DLQ

- **Status:** Aceita
- **Data:** 24/10/2026
- **Decisores:** Larissa, Diego, Bruno ([09:15]–[09:19])

## Contexto

Um cliente pode estar indisponível no momento da entrega (manutenção planejada, erro de rede, timeout). A entrega precisa de uma política de retry que não deixe eventos pendurados para sempre nem desista cedo demais. Já houve cliente com indisponibilidade de duas horas em manutenção planejada ([09:16] Diego). O timeout de cada chamada HTTP é de 10 segundos ([09:42] Diego).

## Decisão

Aplicar **backoff progressivo com 1 tentativa inicial + 5 retentativas**, nos intervalos **1 minuto, 5 minutos, 30 minutos, 2 horas e 12 horas** — total de quase 15 horas entre a primeira falha e a última tentativa ([09:17] Diego, [09:17] Larissa). Esgotadas as tentativas, o evento é movido para uma tabela separada **`webhook_dead_letter`**, contendo payload, motivo da falha e timestamp ([09:18] Diego).

O reprocessamento é **manual**, via endpoint `POST /admin/webhooks/dead-letter/:id/replay`, restrito a **role ADMIN** e **auditado** (log de quem executou) ([09:18] Diego, [09:36] Sofia, [09:36] Larissa).

## Alternativas Consideradas

- **3 tentativas (mais agressivo):** descartado porque três retentativas em ~30 minutos matariam o evento diante de uma indisponibilidade planejada de 2 horas ([09:16] Bruno, [09:16] Diego).
- **Retry infinito com backoff:** descartado porque um cliente que "sumiu" deixaria o evento pendurado para sempre ([09:15] Diego).
- **Marcar como "failed" na própria outbox:** descartado em favor de uma tabela separada, que mantém a leitura da outbox principal limpa e preserva evidência para debug e reprocessamento ([09:18] Diego).

## Consequências

**Positivas**

- Cobre indisponibilidades reais (até ~15h) sem exigir intervenção manual.
- Não deixa eventos órfãos em retry infinito.
- A DLQ separada mantém a outbox principal enxuta e serve de evidência auditável para diagnóstico.

**Negativas**

- Um cliente fora do ar por mais de ~15h depende de replay manual por um ADMIN.
- Aumenta a superfície operacional: mais uma tabela e um fluxo administrativo para monitorar.
- A janela de retry de um evento pode atrasar a entrega de eventos posteriores do mesmo pedido (interação com a ordenação por `order_id`).
