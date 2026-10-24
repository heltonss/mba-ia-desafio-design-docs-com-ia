# ADR-004 — HMAC-SHA256 com secret por endpoint

- **Status:** Aceita
- **Data:** 24/10/2026
- **Decisores:** Sofia, Larissa, Bruno ([09:19]–[09:24])

## Contexto

As notificações expõem dados de pedidos para um endpoint fora da infraestrutura. O cliente precisa validar que a requisição veio do OMS e que o payload não foi adulterado no caminho ([09:19] Sofia). Já houve incidente de cliente vazando secret em log de aplicação ([09:22] Diego), o que reforça a necessidade de segredos isolados e rotacionáveis.

## Decisão

Assinar cada entrega com **HMAC-SHA256 sobre o corpo da requisição**, usando uma secret compartilhada, enviando a assinatura no header **`X-Signature`** ([09:20] Sofia). Detalhes fechados:

- **Secret única por endpoint de webhook** — não uma secret global da plataforma, para limitar o impacto de um vazamento a um único cadastro ([09:21] Sofia).
- A secret é **gerada pela API** e devolvida ao cliente na criação ([09:21] Sofia, [09:31] Marcos).
- **Rotação com grace period de 24 horas:** ao rotacionar, a secret anterior permanece válida em paralelo por 24h para o cliente migrar seus sistemas; depois expira ([09:21] Sofia, [09:22] Sofia).
- **URL somente HTTPS:** URL com `http` é recusada na validação do schema Zod ([09:23] Sofia).
- **Limite de payload de 64KB**, com erro (sem truncamento) acima do limite ([09:23] Sofia, [09:24] Diego, [09:24] Larissa).

## Alternativas Consideradas

- **Secret global única da plataforma:** descartada porque, se vazar uma, vaza tudo ([09:21] Sofia).
- **Sem assinatura (confiar só em HTTPS):** insuficiente, pois HTTPS garante o transporte mas não autentica a origem nem protege contra adulteração no meio do caminho ([09:19] Sofia).
- **Assinatura assimétrica:** não foi discutida; HMAC-SHA256 foi escolhido por ser o padrão de mercado e por todo cliente ter biblioteca pronta ([09:20] Sofia).

## Consequências

**Positivas**

- Cliente consegue autenticar a origem e verificar a integridade do payload.
- Blast radius de vazamento limitado a um endpoint (secret única) e com rotação prevista.
- Padrão interoperável e amplamente suportado.

**Negativas**

- Custo para o cliente: armazenar a secret com segurança e implementar a verificação HMAC.
- A rotação exige uma janela de convivência de 24h com duas secrets válidas; o formato exato de convivência no `X-Signature` ficou em aberto ([09:22] Sofia).
- O HMAC-SHA256 sobre o corpo torna o corpo da requisição um contrato sensível (qualquer mudança no payload invalida a assinatura).

**Observação:** o algoritmo HMAC-SHA256 e o header `X-Signature` são confirmados; o formato exato do valor (hex/base64, com ou sem prefixo) não foi definido na reunião.
