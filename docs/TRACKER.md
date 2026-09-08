# Tracker de Rastreabilidade

Tabela que mapeia cada item registrado nos documentos de design (ADRs) à sua origem real, na
transcrição da reunião (`TRANSCRICAO.md`) ou no código-fonte da aplicação (`src/`). Cobre os 6 ADRs.

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| ADR-001-CTX-01 | docs/adrs/ADR-001-outbox-no-mysql.md | Restrição | Transação de changeStatus já é pesada (order, history, estoque) | TRANSCRICAO | [09:04] Bruno |
| ADR-001-CTX-02 | docs/adrs/ADR-001-outbox-no-mysql.md | Restrição | Não é possível dar rollback de status se cliente estiver fora do ar | TRANSCRICAO | [09:04] Bruno |
| ADR-001-DEC | docs/adrs/ADR-001-outbox-no-mysql.md | Decisão | Outbox em MySQL, evento inserido na mesma transação da mudança de status | TRANSCRICAO | [09:06] Diego |
| ADR-001-ALT-01 | docs/adrs/ADR-001-outbox-no-mysql.md | Alternativa Descartada | Disparo síncrono no service de orders, descartado por travar a transação | TRANSCRICAO | [09:04] Bruno |
| ADR-001-ALT-02 | docs/adrs/ADR-001-outbox-no-mysql.md | Alternativa Descartada | Redis Streams ou fila externa, descartado por overengineering | TRANSCRICAO | [09:07] Diego |
| ADR-001-CONS-01 | docs/adrs/ADR-001-outbox-no-mysql.md | Trade-off | Positiva: atomicidade garantida entre status do pedido e evento | TRANSCRICAO | [09:06] Diego |
| ADR-001-CONS-02 | docs/adrs/ADR-001-outbox-no-mysql.md | Trade-off | Negativa: tabela adicional exige indexação e arquivamento futuro | TRANSCRICAO | [09:08] Diego |
| ADR-001-REF-01 | docs/adrs/ADR-001-outbox-no-mysql.md | Referência | Ponto de integração da outbox no método changeStatus | CODIGO | src/modules/orders/order.service.ts |
| ADR-002-CTX-01 | docs/adrs/ADR-002-worker-separado-com-polling.md | Restrição | Clientes definem "tempo real" como abaixo de 10 segundos | TRANSCRICAO | [09:02] Marcos |
| ADR-002-CTX-02 | docs/adrs/ADR-002-worker-separado-com-polling.md | Restrição | Worker não pode morrer se a API reiniciar | TRANSCRICAO | [09:11] Diego |
| ADR-002-DEC | docs/adrs/ADR-002-worker-separado-com-polling.md | Decisão | Worker em processo separado (src/worker.ts), polling a cada 2 segundos | TRANSCRICAO | [09:10] Larissa |
| ADR-002-ALT-01 | docs/adrs/ADR-002-worker-separado-com-polling.md | Alternativa Descartada | Trigger de banco, descartado por MySQL não ter LISTEN/NOTIFY | TRANSCRICAO | [09:09] Diego |
| ADR-002-ALT-02 | docs/adrs/ADR-002-worker-separado-com-polling.md | Alternativa Descartada | Worker no mesmo processo da API, descartado por não sobreviver a reinício | TRANSCRICAO | [09:11] Diego |
| ADR-002-CONS-01 | docs/adrs/ADR-002-worker-separado-com-polling.md | Trade-off | Positiva: atende requisito de latência abaixo de 10s com folga | TRANSCRICAO | [09:10] Larissa |
| ADR-002-CONS-02 | docs/adrs/ADR-002-worker-separado-com-polling.md | Trade-off | Negativa: sem garantia de ordering global se escalar para múltiplos workers | TRANSCRICAO | [09:12] Diego |
| ADR-002-REF-01 | docs/adrs/ADR-002-worker-separado-com-polling.md | Referência | Padrão de entry-point replicado para o worker | CODIGO | src/server.ts |
| ADR-003-CTX-01 | docs/adrs/ADR-003-retry-com-backoff-e-dead-letter-queue.md | Restrição | Cliente já teve indisponibilidade real de 2h em manutenção planejada | TRANSCRICAO | [09:16] Diego |
| ADR-003-DEC | docs/adrs/ADR-003-retry-com-backoff-e-dead-letter-queue.md | Decisão | 5 tentativas, backoff 1m/5m/30m/2h/12h, depois tabela webhook_dead_letter | TRANSCRICAO | [09:17] Larissa |
| ADR-003-ALT-01 | docs/adrs/ADR-003-retry-com-backoff-e-dead-letter-queue.md | Alternativa Descartada | 3 tentativas, descartado por ser agressivo demais | TRANSCRICAO | [09:16] Bruno |
| ADR-003-ALT-02 | docs/adrs/ADR-003-retry-com-backoff-e-dead-letter-queue.md | Alternativa Descartada | Retry indefinido, descartado por deixar evento pendurado para sempre | TRANSCRICAO | [09:15] Diego |
| ADR-003-ALT-03 | docs/adrs/ADR-003-retry-com-backoff-e-dead-letter-queue.md | Alternativa Descartada | Marcar failed na própria outbox, descartado por poluir a leitura | TRANSCRICAO | [09:18] Diego |
| ADR-003-CONS-01 | docs/adrs/ADR-003-retry-com-backoff-e-dead-letter-queue.md | Trade-off | Positiva: cobre janelas de indisponibilidade reais sem intervenção manual | TRANSCRICAO | [09:17] Marcos |
| ADR-003-CONS-02 | docs/adrs/ADR-003-retry-com-backoff-e-dead-letter-queue.md | Trade-off | Negativa: exige endpoint admin extra com auditoria de replay | TRANSCRICAO | [09:36] Sofia |
| ADR-003-REF-01 | docs/adrs/ADR-003-retry-com-backoff-e-dead-letter-queue.md | Referência | requireRole reaproveitado para restringir replay a ADMIN | CODIGO | src/middlewares/auth.middleware.ts |
| ADR-004-CTX-01 | docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md | Restrição | Cliente precisa validar autenticidade e integridade do payload recebido | TRANSCRICAO | [09:19] Sofia |
| ADR-004-CTX-02 | docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md | Restrição | Incidente anterior de vazamento de secret em log de aplicação de cliente | TRANSCRICAO | [09:22] Diego |
| ADR-004-DEC | docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md | Decisão | HMAC-SHA256 no header X-Signature, secret por endpoint, rotação com grace period de 24h | TRANSCRICAO | [09:22] Sofia |
| ADR-004-ALT-01 | docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md | Alternativa Descartada | Secret global da plataforma, descartada porque vazamento comprometeria todos os clientes | TRANSCRICAO | [09:21] Sofia |
| ADR-004-CONS-01 | docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md | Trade-off | Positiva: vazamento de secret fica isolado a um único endpoint | TRANSCRICAO | [09:21] Sofia |
| ADR-004-CONS-02 | docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md | Trade-off | Negativa: tabela de configuração passa a guardar um dado sensível adicional | TRANSCRICAO | [09:21] Bruno |
| ADR-005-CTX-01 | docs/adrs/ADR-005-garantia-at-least-once-com-event-id.md | Restrição | Garantir exactly-once exigiria coordenação complexa entre os dois lados | TRANSCRICAO | [09:25] Diego |
| ADR-005-DEC | docs/adrs/ADR-005-garantia-at-least-once-com-event-id.md | Decisão | At-least-once com X-Event-Id (UUID) para dedup do lado do cliente | TRANSCRICAO | [09:26] Larissa |
| ADR-005-ALT-01 | docs/adrs/ADR-005-garantia-at-least-once-com-event-id.md | Alternativa Descartada | Exactly-once, descartado por complexidade desproporcional | TRANSCRICAO | [09:25] Diego |
| ADR-005-CONS-01 | docs/adrs/ADR-005-garantia-at-least-once-com-event-id.md | Trade-off | Positiva: solução simples, segue padrão de mercado (Stripe, GitHub) | TRANSCRICAO | [09:25] Diego |
| ADR-005-CONS-02 | docs/adrs/ADR-005-garantia-at-least-once-com-event-id.md | Trade-off | Negativa: joga a responsabilidade de deduplicação para o cliente | TRANSCRICAO | [09:25] Sofia |
| ADR-006-CTX-01 | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Restrição | Projeto já tem estrutura modular consolidada por domínio | TRANSCRICAO | [09:27] Bruno |
| ADR-006-CTX-02 | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Restrição | AppError e subclasses com código próprio já usadas nos módulos existentes | CODIGO | src/shared/errors/http-errors.ts |
| ADR-006-DEC | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Decisão | Módulo webhooks segue estrutura modular, prefixo WEBHOOK_*, reaproveita error middleware, logger e requireRole | TRANSCRICAO | [09:30] Larissa |
| ADR-006-ALT-01 | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Alternativa Descartada | Nenhuma alternativa avaliada; decisão seguiu direto o padrão já existente | TRANSCRICAO | [09:30] Larissa |
| ADR-006-CONS-01 | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Trade-off | Positiva: consistência arquitetural sem introduzir nova lib ou convenção | TRANSCRICAO | [09:30] Larissa |
| ADR-006-CONS-02 | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Trade-off | Negativa: limitações já existentes no error middleware se propagam ao módulo | CODIGO | src/middlewares/error.middleware.ts |
| ADR-006-REF-01 | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Referência | Classe base de erro reaproveitada | CODIGO | src/shared/errors/app-error.ts |
| ADR-006-REF-02 | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Referência | Classes de erro específicas reaproveitadas como padrão | CODIGO | src/shared/errors/http-errors.ts |
| ADR-006-REF-03 | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Referência | Error middleware centralizado reaproveitado sem alteração | CODIGO | src/middlewares/error.middleware.ts |
| ADR-006-REF-04 | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Referência | requireRole reaproveitado para autorização por papel | CODIGO | src/middlewares/auth.middleware.ts |
| ADR-006-REF-05 | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Referência | Logger Pino já configurado, reaproveitado sem nova lib | CODIGO | src/shared/logger/index.ts |
| ADR-006-REF-06 | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Referência | Padrão de módulo (controller/service/repository/routes/schemas) já em uso | CODIGO | src/modules/orders/order.service.ts |
