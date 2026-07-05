# ADR-006: Reuso máximo dos padrões arquiteturais existentes do projeto

## Status

Aceito

## Contexto

O time discutiu explicitamente como estruturar o novo módulo de webhooks dentro da codebase existente, evitando introduzir padrões paralelos ou divergentes dos já estabelecidos no OMS. Bruno descreveu o padrão atual: cada domínio é um módulo em `src/modules/<dominio>` com `controller`, `service`, `repository`, `routes` e `schemas` (ex: `src/modules/customers/`, `src/modules/orders/`) (`[09:27] Bruno`).

Também existe um padrão de tratamento de erros consolidado: a classe base `AppError` (`src/shared/errors/app-error.ts`) e subclasses específicas como `InsufficientStockError` e `InvalidStatusTransitionError` (`src/shared/errors/http-errors.ts`), cada uma com um `errorCode` (ex: `INSUFFICIENT_STOCK`, `INVALID_STATUS_TRANSITION`). O `errorMiddleware` centralizado (`src/middlewares/error.middleware.ts`) já trata instâncias de `AppError`, `ZodError` e erros conhecidos do Prisma sem precisar de alterações (`[09:28]-[09:29] Bruno`). O logger é Pino, já configurado globalmente (`src/shared/logger/index.ts`) (`[09:29] Bruno`).

## Decisão

O módulo de webhooks segue integralmente os padrões já estabelecidos:

- Estrutura de módulo idêntica aos demais: `src/modules/webhooks/` com `webhook.controller.ts`, `webhook.service.ts`, `webhook.repository.ts`, `webhook.routes.ts`, `webhook.schemas.ts` (`[09:27]-[09:28] Bruno`).
- O worker é um entry-point separado (`src/worker.ts`), com a lógica de processamento dentro do módulo (ex: `webhook.processor.ts` ou `webhook.worker.ts`) (`[09:28] Bruno`).
- Erros do módulo estendem `AppError`/as classes de `src/shared/errors/http-errors.ts`, com códigos prefixados por `WEBHOOK_` (ex: `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`, `WEBHOOK_SECRET_REQUIRED`), seguindo o mesmo padrão de `INSUFFICIENT_STOCK` e `INVALID_STATUS_TRANSITION` (`[09:28]-[09:29] Bruno`).
- Nenhuma nova ferramenta de logging é introduzida — reutiliza-se o logger Pino já configurado em `src/shared/logger/index.ts` (`[09:29] Bruno`).
- O `errorMiddleware` existente (`src/middlewares/error.middleware.ts`) trata os erros do módulo de webhooks sem exigir modificação, pois já reconhece qualquer instância de `AppError` (`[09:29] Bruno`).
- Validação de entrada usa o mesmo padrão de schemas Zod + middleware `validate` (`src/middlewares/validate.middleware.ts`) usado pelos demais módulos (evidenciado em `src/modules/customers/customer.schemas.ts` e `src/modules/orders/order.schemas.ts`).
- Autorização reaproveita o middleware `requireRole` existente (`src/middlewares/auth.middleware.ts`) para restringir o endpoint de replay de DLQ à role `ADMIN` (`[09:36] Larissa`).

## Alternativas Consideradas

- **Introduzir uma estrutura própria para o módulo de webhooks** (por exemplo, separando por camadas técnicas em vez de seguir o padrão por domínio): não foi levantada como alternativa real na reunião — a equipe convergiu diretamente para "reuso máximo do que já existe" como decisão explícita (`[09:30] Larissa`), rejeitando implicitamente qualquer desvio do padrão modular estabelecido.
- **Nova biblioteca de logging ou tratamento de erros dedicada ao módulo**: descartada implicitamente ao decidir reaproveitar Pino e `AppError`/error middleware sem alterações (`[09:29] Bruno`).

## Consequências

**Positivas:**
- Redução de curva de aprendizado para qualquer desenvolvedor já familiarizado com os demais módulos do OMS.
- Nenhuma mudança é necessária no `errorMiddleware` central — o módulo de webhooks "encaixa" na infraestrutura de tratamento de erros existente.
- Consistência de código facilita revisão, manutenção e onboarding.

**Negativas:**
- Caso o padrão modular atual tenha limitações (ex: acoplamento de repository ao Prisma), essas limitações são herdadas pelo módulo de webhooks em vez de resolvidas.
- Menor liberdade de design: decisões técnicas do módulo de webhooks ficam restritas ao vocabulário arquitetural já em uso no projeto, mesmo que uma abordagem alternativa pudesse ser marginalmente mais adequada ao caso específico de eventos assíncronos.
