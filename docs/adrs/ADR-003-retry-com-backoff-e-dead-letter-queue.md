# ADR-003: Retry com backoff exponencial e Dead Letter Queue em tabela separada

**Status:** Aceito
**Decisores:** Larissa (Tech Lead), Diego (Engenheiro Sênior, Plataforma), Bruno (Engenheiro Pleno, Pedidos), Sofia (Engenharia de Segurança)
**Relacionado a:** ADR-001, ADR-002

---

## Contexto

Clientes externos podem ficar temporariamente indisponíveis, inclusive por janelas longas: a equipe já teve um cliente com indisponibilidade de duas horas durante uma manutenção planejada. O sistema precisa tentar entregar o evento novamente sem, de um lado, desistir cedo demais em uma indisponibilidade passageira, e sem, de outro lado, deixar o evento tentando ser entregue para sempre caso o cliente tenha desaparecido definitivamente. Também é preciso decidir onde e como as falhas permanentes ficam registradas, e como alguém consegue reprocessá-las manualmente.

## Decisão

Aplicar backoff exponencial com 5 tentativas de entrega, na progressão de 1 minuto, 5 minutos, 30 minutos, 2 horas e 12 horas (totalizando quase 15 horas entre a primeira falha e a última tentativa). Após esgotar as 5 tentativas, o evento é movido para uma tabela separada, `webhook_dead_letter`, contendo a payload, o motivo da falha e o timestamp, em vez de apenas ser marcado como "failed" na própria `webhook_outbox`. O reprocessamento de eventos em dead letter é manual, via endpoint administrativo `POST /admin/webhooks/dead-letter/:id/replay`, que recoloca o evento na outbox como pendente e é restrito a usuários com role `ADMIN`.

## Alternativas Consideradas

### 3 tentativas de retry
Reduzir o número de tentativas para 3, com uma postura mais agressiva de desistência. Descartada porque, em uma indisponibilidade real de algumas horas (como o caso já observado de manutenção planejada de 2 horas), 3 tentativas em uma janela curta (cerca de 30 minutos) já teriam desistido do evento antes do cliente voltar.

### Retry indefinido com backoff
Continuar tentando entregar o evento indefinidamente, sem um teto de tentativas. Descartada porque, se o cliente desapareceu de forma permanente, o evento ficaria pendurado para sempre, sem nunca ser considerado uma falha definitiva.

### Marcar falha permanente como "failed" na própria tabela outbox
Em vez de mover o evento para uma tabela separada, apenas marcar seu status como falho dentro da própria `webhook_outbox`. Descartada porque poluiria a leitura da outbox principal, dificultando a consulta pelo worker aos eventos realmente pendentes, e reduziria a qualidade da tabela como evidência isolada para debug e reprocessamento.

## Consequências

**Positivas**
- Cobre janelas de indisponibilidade reais já observadas na operação (até a ordem de 15 horas) sem exigir intervenção manual constante da equipe.
- Manter a Dead Letter Queue em tabela separada preserva a `webhook_outbox` limpa para o worker de polling e fornece uma fonte isolada de evidência para debug e reprocessamento.

**Negativas**
- Exige a criação de um endpoint administrativo adicional (com controle de acesso e auditoria) só para o fluxo de replay manual.
- Eventos que esgotam as 5 tentativas exigem intervenção manual explícita; não há retry automático além da janela de 15 horas.

## Referências

- `src/middlewares/auth.middleware.ts` (função `requireRole`, reaproveitada para restringir o endpoint de replay a `ADMIN`)
- TRANSCRICAO.md, trecho [09:14] a [09:19]
