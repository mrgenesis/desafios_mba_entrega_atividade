# ADR-006: Reuso dos padrões arquiteturais existentes do projeto no módulo de webhooks

**Status:** Aceito
**Decisores:** Larissa (Tech Lead), Diego (Engenheiro Sênior, Plataforma), Bruno (Engenheiro Pleno, Pedidos)
**Relacionado a:** ADR-002, ADR-003

---

## Contexto

O Order Management System já estabelece convenções consolidadas ao longo dos módulos existentes (auth, users, customers, products, orders): estrutura modular com controller, service, repository, routes e schemas dentro de `src/modules/<dominio>`; tratamento de erro centrado na classe `AppError` e em subclasses específicas com código próprio, como `InsufficientStockError` e `InvalidStatusTransitionError` (`src/shared/errors/http-errors.ts`); um error middleware centralizado que já trata `AppError`, `ZodError` e erros do Prisma sem precisar de alteração por módulo (`src/middlewares/error.middleware.ts`); logging estruturado via Pino já configurado para o projeto inteiro (`src/shared/logger/index.ts`); e um middleware `requireRole` para autorização por papel (`src/middlewares/auth.middleware.ts`). A introdução do módulo de webhooks é uma oportunidade de decidir explicitamente se esses padrões são reaproveitados ou se o novo módulo segue convenções próprias.

## Decisão

O módulo de webhooks segue integralmente os padrões já estabelecidos no projeto: nova pasta `src/modules/webhooks` com a mesma estrutura de controller, service, repository, routes e schemas dos demais módulos; classes de erro específicas do domínio estendendo `AppError`, com códigos usando o prefixo `WEBHOOK_*` (por exemplo `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`, `WEBHOOK_SECRET_REQUIRED`), no mesmo estilo de `INSUFFICIENT_STOCK` e `INVALID_STATUS_TRANSITION`; reaproveitamento do error middleware centralizado sem qualquer alteração, já que ele reconhece `AppError` automaticamente; reaproveitamento do logger Pino já configurado, sem introduzir nova biblioteca de logging; e reaproveitamento do middleware `requireRole` para restringir o endpoint de replay de dead letter (ver ADR-003) à role `ADMIN`. O worker (ver ADR-002) abre sua própria instância de `PrismaClient`, apontando para o mesmo banco (`DATABASE_URL`), respeitando o fato de que `PrismaClient` é uma instância por processo.

## Alternativas Consideradas

### Nenhuma alternativa foi avaliada
A equipe não cogitou introduzir um padrão de estrutura, tratamento de erro, autenticação, autorização ou logging diferente dos já estabelecidos no projeto. A decisão seguiu diretamente o reuso máximo do que já existe, conforme resumido pela tech lead ao final da discussão.

## Consequências

**Positivas**
- Mantém consistência arquitetural entre o módulo de webhooks e os módulos já existentes, reduzindo a curva de aprendizado para qualquer desenvolvedor já familiarizado com o projeto.
- Evita introduzir uma nova biblioteca de logging, um novo padrão de tratamento de erro ou uma nova convenção de estrutura de módulo só para esta feature.

**Negativas**
- Qualquer limitação já existente nesses padrões (por exemplo, os tipos de erro que o error middleware centralizado reconhece hoje: `AppError`, `ZodError` e erros do Prisma) se propaga automaticamente para o módulo de webhooks.
- Reduz a liberdade de adotar uma abordagem potencialmente mais adequada ao domínio específico de webhooks, caso ela divergisse dos padrões já em uso no restante do projeto.

## Referências

- `src/shared/errors/app-error.ts`
- `src/shared/errors/http-errors.ts`
- `src/middlewares/error.middleware.ts`
- `src/middlewares/auth.middleware.ts`
- `src/shared/logger/index.ts`
- `src/modules/orders/order.service.ts` (referência de padrão de módulo já existente)
- TRANSCRICAO.md, trecho [09:27] a [09:30]
