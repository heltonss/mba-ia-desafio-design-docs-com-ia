# Da Reunião ao Documento — Relato do Processo

> Este repositório entrega o pacote de design docs do **Sistema de Webhooks de Notificação de Pedidos**, produzido a partir da transcrição de uma reunião técnica (`TRANSCRICAO.md`) e do código OMS existente. Este `README.md` documenta **como** os documentos foram produzidos. O enunciado original do desafio está em [devfullcycle/mba-ia-desafio-design-docs-com-ia](https://github.com/devfullcycle/mba-ia-desafio-design-docs-com-ia).

---

## Sobre o desafio

A tarefa é transformar a gravação literal de uma reunião de ~55 minutos — na qual tech lead, PM, engenheiros e segurança decidem, mas **não registram**, a arquitetura de uma feature de webhooks — em um pacote de documentação acionável o suficiente para o time de engenharia começar a codar.

O entregável é puramente documental: um `PRD` (por quê/o quê), um `RFC` (arquitetura, em revisão), de 5 a 8 `ADRs` (uma decisão cada), um `FDD` (como construir, em detalhe) e um `TRACKER.md` que liga cada item à sua origem na transcrição ou no código. A regra central do desafio é a rastreabilidade: **nada pode ser inventado**; toda decisão, requisito ou restrição precisa apontar para uma fala (`[hh:mm] Nome`) ou para um arquivo real. O `TRACKER.md` existe justamente para expor — e, na prática, impedir — alucinações da IA.

## Ferramentas de IA utilizadas

- **opencode (CLI/agente)** — ferramenta principal de orquestração. Usei-a para ler a transcrição e o código-fonte, estruturar os documentos, escrevê-los e editá-los iterativamente. Foi o "maestro": eu definia o que precisava ser feito, e ela executava leitura, busca e escrita dentro do repositório.
- **Modelo de linguagem (deepseek-v4-pro, via opencode)** — motor de geração de conteúdo. Responsável por interpretar a transcrição, mapear pontos de integração no código e redigir o texto dos cinco documentos, sempre sob revisão crítica minha.
- **Ferramentas de busca/leitura do agente (glob, grep, read)** — usadas para localizar os pontos reais de integração no código (ex.: a transação de `changeStatus`, as classes de erro, o middleware de auth) antes de citá-los nos documentos.
- **git** — usado para revisar o histórico do repositório (incluindo o enunciado original do desafio, recuperado do commit inicial) e versionar cada ciclo de ajuste.

Não foram usados plugins, skills externos ou geração via web; todo o trabalho partiu da transcrição e do código local.

## Workflow adotado

A ordem de produção seguiu a sugestão do enunciado, com os **ADRs primeiro**, porque as decisões são o esqueleto do "como implementar":

1. **Contextualização** — li `TRANSCRICAO.md` inteira e explorei o código (`src/modules/orders`, `src/shared/errors`, `src/middlewares`, `src/app.ts`, `prisma/schema.prisma`), montando o mapa dos pontos de integração reais.
2. **ADRs** — extraí as decisões fechadas e as transformei em 6 ADRs (`ADR-001` a `ADR-006`), cobrindo as 6 decisões principais exigidas.
3. **RFC** — consolidei a proposta técnica em cima das ADRs, colocando alternativas descartadas e questões em aberto no lugar natural.
4. **FDD** — com as decisões formalizadas, detalhei fluxos, contratos, matriz de erros `WEBHOOK_*`, observabilidade e a seção "Integração com o sistema existente".
5. **PRD** — por último, como consolidação de mais alto nível (problema, escopo, requisitos, métricas, riscos).
6. **Tracker** — varri os documentos prontos e montei a referência cruzada, removendo os itens sem origem identificável.
7. **Validação final** — rodei uma checagem minuciosa contra os critérios de aceite do desafio (seções obrigatórias, contagens mínimas, percentuais do tracker, caminhos de código citados) e corrigi as pendências encontradas antes de fechar.
8. **README** — este relato, ao final, quando o processo já estava completo.

A interação com a IA foi sempre dirigida: cada documento nasceu de um prompt específico (não de um pedido genérico "gere o PRD"), e cada saída passou por revisão crítica antes de ser aceita. A validação final foi feita com o mesmo rigor: a IA leu cada arquivo entregue, confrontou com a transcrição e com o código, e reportou divergências que eu então corrigi.

## Prompts customizados

Dois prompts que dirigiram o trabalho (adaptados do contexto do curso):

**1. Contextualização e extração de pontos de integração (antes de escrever qualquer doc):**

```text
Aja como engenheiro revisando um OMS Node.js/TypeScript + Prisma + MySQL.
Leia TRANSCRICAO.md e o código em src/ e prisma/ e me devolva:

1. Lista das DECISÕES FECHADAS da reunião (uma linha cada), com o
   timestamp [hh:mm] Nome que a sustenta.
2. Lista do que foi EXPLICITAMENTE DESCARTADO ou ADIADO.
3. Os pontos REAIS de integração com o código: caminhos de arquivo
   existentes e o que cada um faz (ex.: onde começa a transação de
   changeStatus, quais classes de erro existem, onde fica o requireRole).
4. Liste também o que NÃO existe no código (worker, fila, webhook, outbox).

Não invente fatos. Se algo não estiver na transcrição ou no código,
marque explicitamente como "não encontrado".
```

**2. Geração do FDD com marcação obrigatória de confirmação vs. hipótese:**

```text
Escreva docs/FDD.md para a feature de webhooks. Regras:

- Apenas informação rastreável a TRANSCRICAO.md ou ao código existente.
- Para cada decisão técnica, cite a fala que a sustenta no formato
  [hh:mm] Nome. O que a transcrição NÃO define deve ser rotulado como
  "proposta", "hipótese" ou "inferência" — nunca como decisão fechada.
- Inclua a seção "Integração com o sistema existente" citando pelo
  menos 4 caminhos REAIS de arquivo e a linha relevante.
- Separe em "Incluído" e "Excluído" os itens de escopo, espelhando o que
  a reunião descartou (síncrono, Redis, trigger, exactly-once, email...).
- Não repita o nível de detalhe do RFC; o FDD responde "como construir".
```

**3. Varredura anti-alucinação para o tracker:**

```text
Monte docs/TRACKER.md varrendo PRD, RFC, FDD e ADRs. Para cada item,
preencha Fonte (TRANSCRICAO ou CODIGO) e Localização ([hh:mm] Nome ou
caminho real). Se você não conseguir preencher "Localização" de uma
linha, NÃO crie a linha — isso indica conteúdo inventado. Marque como
CODIGO apenas caminhos de arquivos que existem de fato no repositório.
```

**4. Validação contra os critérios de aceite (usado no ciclo final):**

```text
Faça uma busca minuciosa na entrega e valide contra os Critérios de
Aceite do desafio, item a item. Para cada critério, responda PASSA ou
FALHA com evidência (arquivo, seção, contagem, linha). Verifique também:
1. Todo caminho de código citado nos documentos existe no repositório,
   na linha indicada.
2. Nenhuma decisão registrada contradiz TRANSCRICAO.md ou o código.
3. Os percentuais do TRACKER.md (cobertura, Fonte=TRANSCRICAO,
   Fonte=CODIGO) e os mínimos numéricos dos documentos.
Liste observações menores que não bloqueiam, para que eu decida corrigir.
```

## Iterações e ajustes

A produção levou cerca de **4 ciclos principais** de geração → revisão → correção, mais **1 ciclo de validação final** contra os critérios de aceite. Os momentos em que a IA errou ou ficou superficial, e que exigiram correção:

1. **Documentos genéricos e com duplicação.** As primeiras versões repetiam a mesma decisão nas quatro alturas (PRD/RFC/FDD/ADRs) e soavam como cópia da transcrição. Ajustei os prompts para reforçar a fronteira entre os documentos (RFC = decisão, FDD = implementação) e pedir remoção de trechos vagos.
2. **Confusão entre "5 tentativas" e a contagem de chamadas.** A transcrição diz "5 tentativas" mas lista 5 intervalos de backoff (1m/5m/30m/2h/12h), que só fecham ~15h se contarmos 1 tentativa inicial + 5 retentativas (6 chamadas). A IA registrou a leitura correta, mas precisei garantir que a divergência ficasse documentada explicitamente (está no fluxo 4 e no critério 13 do FDD) em vez de escondida.
3. **`customerId` vindo do JWT.** A sugestão inicial da reunião foi corrigida na própria call: o `customerId` deve ser explícito no body/path, porque o JWT representa o usuário operador, não o cliente. A IA inicialmente propagou a versão errada; corrigi para refletir a decisão final ([09:32] Larissa).
4. **Hipóteses vendidas como fatos.** Valores como percentis de latência, disponibilidade 99,9%, tamanho do lote do worker e formato exato do `X-Signature` não foram decididos na reunião. A IA os apresentou como requisitos; corrigi para marcá-los como "hipótese a validar" em todo o pacote, mantendo apenas o que é rastreável como decisão.
5. **Ciclo de validação final.** Na checagem contra os critérios de aceite, a IA apontou 4 pendências menores, que corrigi antes de fechar: o PRD não tinha subseção explícita de "Cenários de uso" (adicionada com 4 cenários rastreáveis); o RFC listava 4 revisores em vez dos 5 participantes da reunião; `docs/HLD.md` era um arquivo extra fora da estrutura obrigatória (removido, e o cabeçalho do FDD ajustado); e a menção a `src/worker.ts` no RFC não marcava o arquivo como novo (marcada).

O `TRACKER.md` foi o instrumento que tornou essas correções visíveis: toda linha sem `Localização` válida era sinal de invenção e foi ajustada ou removida. No estado final, o tracker tem 134 linhas — 113 com fonte na transcrição (84,3%) e 21 com fonte no código real — e todos os caminhos de arquivo citados nos documentos existem no repositório, nas linhas indicadas.

## Como navegar a entrega

Arquivos entregues (apenas `docs/` e este `README.md` foram alterados; `src/`, `prisma/`, `tests/` e `TRANSCRICAO.md` permanecem intactos):

```
.
├── README.md                  ← este relato do processo
├── TRANSCRICAO.md             ← fonte primária (não alterar)
└── docs/
    ├── PRD.md                 ← por quê / o quê (produto e negócio)
    ├── RFC.md                 ← arquitetura proposta, em revisão
    ├── FDD.md                 ← como construir, em detalhe
    ├── TRACKER.md             ← rastreabilidade de cada item
    └── adrs/
        ├── ADR-001-outbox-transacional-no-mysql.md
        ├── ADR-002-worker-em-processo-separado-com-polling.md
        ├── ADR-003-retry-com-backoff-e-dlq.md
        ├── ADR-004-hmac-sha256-com-secret-por-endpoint.md
        ├── ADR-005-entrega-at-least-once-com-x-event-id.md
        └── ADR-006-reuso-dos-padroes-existentes-do-projeto.md
```

Ordem sugerida de leitura:

1. `docs/PRD.md` — entender o problema, os cenários de uso e o escopo.
2. `docs/adrs/*` — as decisões fechadas, uma a uma.
3. `docs/RFC.md` — a proposta consolidada e o que ficou em aberto.
4. `docs/FDD.md` — o detalhe de implementação e a integração com o código.
5. `docs/TRACKER.md` — conferir de onde veio cada item (anti-alucinação).
6. `TRANSCRICAO.md` — fonte primária, para auditar a fidelidade.
