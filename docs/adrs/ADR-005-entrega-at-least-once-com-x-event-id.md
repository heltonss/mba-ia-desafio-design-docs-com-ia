# ADR-005 — Entrega at-least-once com X-Event-Id

- **Status:** Aceita
- **Data:** 24/10/2026
- **Decisores:** Diego, Larissa ([09:24]–[09:26])

## Contexto

Entre o worker e o cliente podem ocorrer duplicatas: por exemplo, o worker recebe 2xx mas cai antes de marcar o evento como entregue, e na retomada reenvia. É preciso definir a garantia de entrega e como o cliente lida com eventos repetidos.

## Decisão

Adotar **entrega at-least-once**: o cliente pode receber o mesmo evento mais de uma vez e deve estar preparado. Para permitir deduplicação, cada evento carrega no header **`X-Event-Id`** um **UUID único gerado quando o evento entra na outbox** ([09:25] Diego). O `X-Event-Id` é estável entre retries e replay, permitindo que o cliente deduplique pelo identificador ([09:25] Diego).

A responsabilidade de idempotência fica, em parte, com o cliente, o que é o padrão de mercado (Stripe, GitHub) ([09:25] Diego). O Marcos se compromete a documentar isso com destaque no portal do desenvolvedor ([09:26] Marcos).

## Alternativas Consideradas

- **Entrega exactly-once:** descartada porque exigiria coordenação dos dois lados (OMS + cliente) e fica muito mais complexa, sem retorno proporcional ([09:25] Diego).
- **At-most-once (sem retry):** não atenderia ao requisito de confiabilidade e recuperação de falhas; a política de retry do ADR-003 depende da possibilidade de reentrega.

## Consequências

**Positivas**

- Simplicidade: não há protocolo de coordenação distribuída nem de deduplicação no lado do OMS.
- Recuperação confiável: retries e replay podem reenviar sem risco de perda silenciosa.
- Padrão de mercado, familiar aos clientes e às bibliotecas de integração.

**Negativas**

- O cliente pode processar o mesmo evento mais de uma vez e precisa implementar deduplicação pelo `X-Event-Id`.
- A responsabilidade de idempotência é parcialmente transferida ao cliente, que precisa ser bem documentado.
- Duplicatas não tratadas pelo cliente podem gerar efeitos duplicados no sistema dele.
