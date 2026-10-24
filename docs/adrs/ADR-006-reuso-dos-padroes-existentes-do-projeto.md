# ADR-006 — Reuso dos padrões existentes do projeto

- **Status:** Aceita
- **Data:** 24/10/2026
- **Decisores:** Larissa, Bruno, Diego ([09:27]–[09:30])

## Contexto

O OMS já tem uma arquitetura modular e padrões consolidados: cada domínio vive em `src/modules` com controller, service, repository, routes e schemas; há classes de erro com códigos estáveis; logger Pino; middleware de erro centralizado; e middleware de autorização. A nova feature de webhooks precisa se encaixar nesses padrões sem reinventar a base.

## Decisão

**Reutilizar ao máximo os padrões e componentes existentes**, sem introduzir bibliotecas novas:

- Criar o módulo **`src/modules/webhooks`** seguindo a estrutura de módulos existente (controller, service, repository, routes e schemas) ([09:27] Bruno).
- Usar **`AppError`** e classes específicas com códigos estáveis, seguindo o padrão de `InsufficientStockError`/`InvalidStatusTransitionError`, com prefixo **`WEBHOOK_`** (ex.: `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`, `WEBHOOK_SECRET_REQUIRED`) ([09:28] Bruno, [09:29] Larissa).
- Manter o logger **Pino** já presente no projeto ([09:29] Bruno).
- Aproveitar o **middleware de erro centralizado**, que já trata `AppError`, Zod e Prisma, sem precisar alterá-lo ([09:29] Bruno).
- Reaproveitar **`requireRole`** para restringir o replay de DLQ a ADMIN ([09:36] Larissa).
- O worker usa o mesmo banco e `DATABASE_URL`, mas com um **PrismaClient separado por processo** ([09:30] Bruno).

## Alternativas Consideradas

- **Criar um serviço independente com stack própria:** descartado implicitamente — o time é pequeno e o reuso mantém consistência, reduz código novo e faz o tratamento de erros funcionar sem mudanças no middleware central ([09:30] Larissa).
- **Introduzir bibliotecas novas de log, erro ou validação:** descartado; o projeto já tem Pino, `AppError`, Zod e o middleware de erro cobrindo o necessário ([09:29] Bruno).

## Consequências

**Positivas**

- Consistência com o resto da aplicação e curva de aprendizado menor para o time.
- Menos código novo e menor superfície de bugs; erros do módulo caem no tratamento centralizado existente.
- Autorização de ADMIN no replay reutiliza `requireRole` sem lógica adicional.

**Negativas**

- O módulo de webhooks fica acoplado ao padrão do monólito e ao mesmo ciclo de deploy da API, em vez de evoluir como serviço independente.
- Herda as limitações do middleware e da estrutura existentes (ex.: sem isolamento de authorization por customer nesta fase).

**Referências de código:** `src/shared/errors/app-error.ts` e `src/shared/errors/http-errors.ts` (padrão de `AppError` e códigos estáveis), `src/middlewares/auth.middleware.ts` (`authenticate` e `requireRole`), `src/middlewares/error.middleware.ts` (tratamento centralizado), `src/shared/logger/index.ts` (Pino), `src/routes/index.ts` e `src/app.ts` (montagem das rotas sob `/api/v1`).
