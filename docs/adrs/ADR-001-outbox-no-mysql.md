# ADR-001: Padrão Outbox no MySQL para entrega de eventos de webhook

**Status:** Aceito
**Decisores:** Larissa (Tech Lead), Diego (Engenheiro Sênior, Plataforma), Bruno (Engenheiro Pleno, Pedidos)

---

## Contexto

O Sistema de Webhooks precisa notificar clientes externos quando o status de um pedido muda, mas a transação de mudança de status já é pesada: ela atualiza a tabela `orders`, insere um registro em `order_status_history` e ajusta `stock_quantity` dos produtos do pedido. Disparar a chamada HTTP de notificação de forma síncrona, dentro dessa mesma transação, arriscaria travar a mudança de status de outros pedidos caso o cliente externo esteja lento, e não existe uma forma sensata de fazer rollback do status caso o cliente esteja fora do ar. É preciso garantir que, se a transação de mudança de status foi commitada, o evento de notificação exista de forma confiável, sem acoplar a disponibilidade do cliente externo à transação principal.

## Decisão

Adotar o padrão Outbox sobre o MySQL já existente no projeto: quando o status do pedido muda, a mesma transação SQL que atualiza `orders` e `order_status_history` também insere uma linha em uma nova tabela `webhook_outbox` com o evento a ser entregue. Um worker separado (ver ADR-002) lê essa tabela de forma assíncrona e realiza as chamadas HTTP. Se a transação principal falhar e sofrer rollback, o evento correspondente desaparece junto, sem inconsistência possível entre o estado do pedido e os eventos pendentes de entrega.

## Alternativas Consideradas

### Disparo síncrono da chamada HTTP dentro do service de orders
Fazer a chamada ao endpoint do cliente diretamente dentro da transação de `changeStatus`. Descartada porque qualquer cliente lento travaria a mudança de status de outros pedidos, e não haveria como reverter a mudança de status caso a chamada falhasse.

### Fila externa dedicada (ex. Redis Streams)
Introduzir uma fila de mensageria fora do MySQL para desacoplar a geração do evento do seu processamento. Descartada porque exigiria subir e operar uma nova peça de infraestrutura (ex. um cluster Redis) para um time pequeno, sendo considerada overengineering frente ao volume esperado; o MySQL já existente resolve o mesmo problema sem infraestrutura adicional.

## Consequências

**Positivas**
- Garante atomicidade entre a mudança de status do pedido e o registro do evento de notificação: o evento só existe se a transação principal foi commitada.
- Reaproveita a infraestrutura de banco de dados já existente (MySQL via Prisma), sem exigir novo componente de infraestrutura.

**Negativas**
- Introduz uma tabela adicional (`webhook_outbox`) que precisa de estratégia própria de indexação e, no futuro, de arquivamento das linhas já entregues (fora do escopo desta feature).
- Acopla a confiabilidade da entrega ao desempenho do MySQL em vez de um sistema de mensageria dedicado, o que pode se tornar uma limitação caso o volume de eventos cresça muito.

## Referências

- `src/modules/orders/order.service.ts` (método `changeStatus`, ponto de integração da inserção na outbox)
- TRANSCRICAO.md, trecho [09:03] a [09:08]
