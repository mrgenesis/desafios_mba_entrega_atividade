# RFC: Sistema de Webhooks de Notificação de Pedidos

**Autor:** Larissa (Tech Lead)
**Status:** Em revisão
**Data:** 2026-09-07
**Versão:** 1.0
**Revisores:** Marcos (Product Manager), Bruno (Engenheiro Pleno, Pedidos), Diego (Engenheiro Sênior, Plataforma), Sofia (Engenharia de Segurança)

---

## Resumo executivo (TL;DR)

Três clientes B2B (Atlas Comercial, MaxDistribuição e Nova Cargo) pediram para ser notificados em tempo real quando o status de um pedido muda, em vez de continuar fazendo polling em `GET /orders`. Propomos um pipeline assíncrono baseado no padrão Outbox sobre o MySQL já existente, com um worker em processo separado entregando os eventos via HTTP, autenticado por HMAC-SHA256 e com garantia de entrega at-least-once. A mudança de status continua síncrona e isolada da disponibilidade dos clientes externos; a notificação passa a ser um efeito colateral confiável, mas assíncrono, dessa transação. As principais decisões arquiteturais já têm consenso da equipe (registradas nos ADRs referenciados); esta RFC consolida a proposta e expõe os pontos que ainda dependem de observação ou decisão futura, como rate limiting de envio e alertas de falha ao cliente.

## Contexto e problema

Hoje os clientes integrados fazem polling periódico em `GET /orders` para saber se o status de um pedido mudou, o que a Atlas, a MaxDistribuição e a Nova Cargo relataram formalmente como lento e caro do lado deles. Para essas empresas, qualquer notificação entregue em menos de 10 segundos já atende à expectativa de "tempo real"; o requisito não é latência mínima agressiva, é eliminar o polling manual. A pressão comercial é concreta: a Atlas sinalizou que pode migrar para um concorrente se a feature não for entregue até o fim do trimestre.

O sistema hoje não tem nenhum mecanismo de notificação externa, eventos ou filas: essa é a lacuna que a feature preenche. A restrição técnica central vem do próprio fluxo de mudança de status: o método `changeStatus` do serviço de pedidos (`src/modules/orders/order.service.ts`) já executa, numa única transação, a atualização de `orders`, a inserção em `order_status_history` e o ajuste de `stock_quantity` dos produtos do pedido. Acoplar uma chamada HTTP síncrona a essa transação criaria dois problemas: um cliente externo lento travaria a mudança de status de outros pedidos, e não existe uma forma sensata de fazer rollback do status caso o cliente esteja fora do ar. Qualquer proposta de solução precisa manter a mudança de status desacoplada da disponibilidade dos sistemas externos.

O escopo desta feature é exclusivamente outbound: o sistema envia notificações para os clientes, os clientes não enviam nada de volta para o sistema.

## Proposta técnica

A proposta adota o padrão Outbox: dentro da mesma transação SQL que já atualiza `orders` e `order_status_history`, o serviço de pedidos insere um evento numa nova tabela `webhook_outbox`, através de uma função (`publishWebhookEvent`) que recebe o cliente de transação (`tx`) em uso. Se a transação principal falhar, o evento correspondente nunca chega a existir; se ela for commitada, o evento fica garantido. Essa decisão e a alternativa de disparo síncrono descartada estão detalhadas em [ADR-001](adrs/ADR-001-outbox-no-mysql.md).

Um processo Node separado (`src/worker.ts`, novo entry-point ao lado de `src/server.ts`) faz polling da tabela `webhook_outbox` a cada 2 segundos, processa os eventos pendentes mais antigos e realiza as chamadas HTTP para os endpoints cadastrados pelos clientes. Rodar em processo separado garante que um reinício da API não interrompa o processamento de eventos pendentes. Ver [ADR-002](adrs/ADR-002-worker-separado-com-polling.md).

Falhas de entrega são tratadas com retry em backoff exponencial (5 tentativas, de 1 minuto a 12 horas). Esgotadas as tentativas, o evento é movido para uma tabela `webhook_dead_letter` separada, reprocessável manualmente por um endpoint administrativo restrito a usuários com role `ADMIN`. Ver [ADR-003](adrs/ADR-003-retry-com-backoff-e-dead-letter-queue.md).

Cada evento é assinado com HMAC-SHA256 usando uma secret exclusiva do endpoint de destino (não uma secret global da plataforma), com suporte a rotação com grace period. Ver [ADR-004](adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md).

A entrega segue a garantia at-least-once: o mesmo evento pode chegar duplicado ao cliente, que é responsável por deduplicar usando um identificador único de evento enviado em cada chamada. Ver [ADR-005](adrs/ADR-005-garantia-at-least-once-com-event-id.md).

O novo módulo `src/modules/webhooks` segue integralmente a estrutura já usada nos módulos existentes do projeto (controller, service, repository, routes, schemas), reaproveita `AppError` e o error middleware centralizado, o logger Pino já configurado e o middleware `requireRole` para autorização. Ver [ADR-006](adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md).

Na superfície de API, a proposta inclui: CRUD de configuração de webhook por cliente (cadastro com URL e filtro de status de interesse, edição, remoção e listagem), um endpoint de consulta ao histórico de entregas de um webhook, e o endpoint administrativo de replay de eventos em dead letter. O filtro de eventos por status é aplicado no momento da inserção na outbox, não no momento do envio: se nenhum webhook do cliente está interessado naquele status, o evento nem é inserido. O detalhamento de contratos, payloads, headers e códigos de erro fica no FDD, não nesta RFC.

## Alternativas consideradas

### Disparo síncrono da notificação dentro da transação de `changeStatus`
Chamar o endpoint do cliente diretamente durante a transação de mudança de status, sem outbox nem processamento assíncrono. Descartada porque um cliente externo lento travaria a mudança de status de outros pedidos, e não haveria como reverter a mudança de status se o cliente estivesse indisponível.

### Fila de mensageria dedicada (ex. Redis Streams)
Introduzir uma fila fora do MySQL para desacoplar a geração do evento do seu processamento. Descartada por exigir subir e operar uma nova peça de infraestrutura para um time pequeno, considerado overengineering frente ao volume esperado; o MySQL já existente resolve o mesmo problema sem infraestrutura adicional.

### Trigger de banco de dados para acionar o worker
Usar um trigger do MySQL para notificar o worker assim que um evento é inserido, tornando o processamento reativo em vez de por polling. Descartada porque o MySQL não tem um mecanismo equivalente ao `LISTEN`/`NOTIFY` do Postgres; um trigger só executa SQL, não notifica um processo externo, e contornar isso exigiria soluções frágeis como escrita em arquivo.

### Garantia de entrega exactly-once
Garantir que cada evento seja entregue exatamente uma vez, sem duplicações possíveis. Descartada porque exigiria coordenação adicional entre os dois lados (empresa e cliente), aumentando muito a complexidade da solução para um ganho que não justifica o esforço frente ao padrão de mercado (at-least-once com identificador de evento, como Stripe e GitHub praticam).

## Questões em aberto

- Alertar o cliente (por exemplo, por e-mail) quando o webhook dele acumula falhas consecutivas de entrega: ficou fora do escopo desta fase, possivelmente para uma fase futura, depois de medido o impacto real.
- Rate limiting de envio para um mesmo cliente quando muitos pedidos mudam de status em um curto intervalo: não entra no escopo atual; a decisão foi observar se isso se torna um problema real antes de implementar qualquer controle.
- Estratégia para preservar a ordenação de eventos por pedido caso o sistema precise escalar para múltiplos workers em paralelo (hoje a ordenação depende de um único worker processando em ordem de criação): tratado como limitação conhecida e problema a resolver apenas se e quando a escala exigir.

## Impacto e riscos

- **Módulo de pedidos:** o método `changeStatus` (`src/modules/orders/order.service.ts`) precisa ser estendido para publicar o evento de webhook dentro da mesma transação, via uma função que recebe o cliente de transação em uso.
- **Infraestrutura/operação:** introduz um novo processo de longa duração (`src/worker.ts`) que precisa ser mantido no ar de forma independente da API, com sua própria instância de `PrismaClient` apontando para o mesmo banco.
- **Segurança:** a Sofia reservou pelo menos dois dias úteis de revisão dedicada ao código de geração e verificação de HMAC e de secrets antes do deploy; esse é um risco de cronograma se a revisão apontar mudanças estruturais.
- **Risco comercial:** a Atlas condicionou a continuidade da parceria à entrega até o fim do trimestre; atraso na entrega tem impacto direto de churn.
- **Risco de disponibilidade de cliente externo:** clientes podem ficar indisponíveis por janelas longas (já houve caso de 2 horas por manutenção planejada); a janela de retry de até ~15 horas antes da dead letter mitiga isso, mas eventos que excedem essa janela exigem reprocessamento manual.
- **Crescimento de dados:** a tabela `webhook_outbox` acumula eventos entregues ao longo do tempo; a estratégia de arquivamento dessas linhas fica fora do escopo desta feature.

## Decisões relacionadas

- [ADR-001: Padrão Outbox no MySQL para entrega de eventos de webhook](adrs/ADR-001-outbox-no-mysql.md)
- [ADR-002: Worker em processo separado com polling de 2 segundos](adrs/ADR-002-worker-separado-com-polling.md)
- [ADR-003: Retry com backoff exponencial e Dead Letter Queue em tabela separada](adrs/ADR-003-retry-com-backoff-e-dead-letter-queue.md)
- [ADR-004: Autenticação HMAC-SHA256 com secret por endpoint e rotação com grace period](adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md)
- [ADR-005: Garantia at-least-once com idempotência via X-Event-Id](adrs/ADR-005-garantia-at-least-once-com-event-id.md)
- [ADR-006: Reuso dos padrões arquiteturais existentes do projeto no módulo de webhooks](adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md)
