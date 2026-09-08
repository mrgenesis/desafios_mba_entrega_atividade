# ADRs (Architecture Decision Records)

Este diretório registra as decisões arquiteturais fechadas da feature de Sistema de Webhooks de
Notificação de Pedidos, extraídas da reunião técnica registrada em `TRANSCRICAO.md` e do código-fonte
existente em `src/`. Cada ADR cobre uma única decisão, com contexto, alternativas descartadas e
consequências (positivas e negativas). Rastreabilidade completa de cada item para a transcrição ou o
código em [`docs/TRACKER.md`](../TRACKER.md).

| ADR | Decisão | Status |
| --- | --- | --- |
| [ADR-001](ADR-001-outbox-no-mysql.md) | Padrão Outbox no MySQL para entrega de eventos de webhook | Aceito |
| [ADR-002](ADR-002-worker-separado-com-polling.md) | Worker em processo separado com polling de 2 segundos | Aceito |
| [ADR-003](ADR-003-retry-com-backoff-e-dead-letter-queue.md) | Retry com backoff exponencial e Dead Letter Queue em tabela separada | Aceito |
| [ADR-004](ADR-004-hmac-sha256-com-secret-por-endpoint.md) | Autenticação HMAC-SHA256 com secret por endpoint e rotação com grace period | Aceito |
| [ADR-005](ADR-005-garantia-at-least-once-com-event-id.md) | Garantia at-least-once com idempotência via X-Event-Id | Aceito |
| [ADR-006](ADR-006-reuso-dos-padroes-existentes-do-projeto.md) | Reuso dos padrões arquiteturais existentes do projeto no módulo de webhooks | Aceito |

## Convenção de numeração

Novos ADRs são numerados sequencialmente a partir do maior `NNN` já existente nesta pasta, no formato
`ADR-NNN-titulo-em-kebab-case.md`. Nenhum número existente é reaproveitado ou sobrescrito.
