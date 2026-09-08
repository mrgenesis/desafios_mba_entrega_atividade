# PRD — Product Requirements Document

### PRD: Order Management System (OMS) Sistema de Webhooks de Notificação de Pedidos

Versão: 1.0
Data: 2026-09-07
Responsável: Marcos (Product Manager)

---

### Resumo

O Order Management System (OMS) hoje não tem nenhum mecanismo de notificação externa, eventos ou filas: clientes B2B integrados descobrem mudanças de status de pedido fazendo polling periódico em `GET /orders`. Três clientes (Atlas Comercial, MaxDistribuição e Nova Cargo) pediram formalmente para serem notificados automaticamente quando o status de um pedido muda, e a Atlas condicionou a continuidade da parceria à entrega até o fim do trimestre. Esta feature adiciona um Sistema de Webhooks de Notificação de Pedidos: os clientes cadastram um endpoint HTTPS, escolhem quais mudanças de status querem observar, e passam a receber notificações assinadas automaticamente, com garantia de entrega at-least-once, dentro de um teto de 10 segundos no cenário sem falhas.

---

### Contexto e problema

Público-alvo
- Clientes B2B integrados ao OMS (Atlas Comercial, MaxDistribuição, Nova Cargo), donos dos endpoints de webhook cadastrados.
- Usuários autenticados do OMS que operam o CRUD de configuração de webhook em nome de um customer (a integração é feita via API autenticada com JWT do OMS, não diretamente pelo cliente externo).
- Usuários com role `ADMIN`, responsáveis pelo reprocessamento manual de eventos que falharam permanentemente.

Cenários de uso chave
- Um cliente B2B cadastra um webhook filtrando pelos status de interesse (por exemplo, `SHIPPED` e `DELIVERED`) e passa a ser notificado automaticamente a cada mudança de status relevante de seus pedidos, sem precisar mais fazer polling em `GET /orders`.
- Um cliente B2B consulta o histórico das últimas entregas de um webhook para auditar se as notificações estão chegando e diagnosticar falhas.
- Um cliente B2B rotaciona a secret de um webhook por política própria de segurança, sem perder notificações durante a janela de transição.
- Um usuário `ADMIN` reprocessa manualmente um evento que ficou em dead letter, por exemplo depois que o endpoint do cliente volta a responder após uma indisponibilidade prolongada.

Onde essa feature será implantada
- Sistema já existente: o OMS (Node.js + TypeScript, Prisma sobre MySQL), com módulos de autenticação, usuários, clientes, produtos e pedidos já em produção. A feature adiciona um novo módulo (`src/modules/webhooks`) e um novo processo de longa duração (`src/worker.ts`), rodando ao lado da API existente.

Problemas priorizados
- Clientes B2B fazem polling periódico em `GET /orders` para saber se o status de um pedido mudou, o que a Atlas, a MaxDistribuição e a Nova Cargo relataram formalmente como lento e caro do lado deles ([09:00] Marcos). Impacto: risco comercial direto, a Atlas sinalizou que pode migrar para um concorrente se a feature não for entregue até o fim do trimestre ([09:00] Marcos). A transcrição não traz uma estimativa numérica de custo ou tempo perdido pelo polling; o impacto documentado é qualitativo (risco de churn).
- O sistema hoje não tem nenhum mecanismo de notificação externa, eventos ou filas: é um vácuo proposital, não uma tentativa anterior malsucedida (não há nada que já tenha sido tentado e descartado).

---

### Objetivos e métricas

| Objetivo | Métrica | Meta |
| --- | --- | --- |
| Eliminar a dependência de polling manual em `GET /orders` para saber quando o status de um pedido muda | Latência entre a mudança de status e a tentativa de entrega da notificação | Abaixo de 10 segundos no cenário sem falhas, com piso de 2 segundos definido pelo ciclo de polling do worker ([09:02] Marcos; [09:09]-[09:10] Diego, Larissa) |
| Garantir que nenhuma notificação de mudança de status se perca silenciosamente | Percentual de eventos elegíveis que chegam a um estado final rastreável (entregues com sucesso ou registrados em dead letter para reprocessamento manual) | 100% dos eventos elegíveis: nenhum evento fica pendurado indefinidamente, ele é entregue ou vai para dead letter dentro da janela de retry de aproximadamente 15 horas (ADR-003; [09:15]-[09:18] Diego) |

---

### Escopo

Incluso
- CRUD de configuração de webhook por customer: cadastro (URL + status observados, secret gerada na criação), edição, remoção e listagem ([09:31]-[09:33] Marcos, Bruno).
- Registro do evento de notificação em uma tabela outbox, dentro da mesma transação da mudança de status, com filtro de status aplicado na inserção ([09:33]-[09:34] Marcos, Bruno, Diego).
- Processamento assíncrono por um worker em processo separado, com polling a cada 2 segundos ([09:10] Larissa).
- Retry com backoff exponencial (5 tentativas) e Dead Letter Queue em tabela separada, com endpoint administrativo de replay manual restrito a role `ADMIN` ([09:15]-[09:19] Diego, Larissa, Sofia).
- Consulta ao histórico de entregas de um webhook, com os últimos 100 registros ([09:34] Marcos).
- Assinatura HMAC-SHA256 por evento, secret exclusiva por endpoint com suporte a rotação e grace period de 24 horas, validação de URL somente HTTPS e limite de 64KB no payload do evento ([09:19]-[09:24] Sofia, Diego, Larissa).
- Garantia de entrega at-least-once, com `X-Event-Id` único por evento para deduplicação do lado do cliente ([09:24]-[09:26] Diego, Sofia, Larissa).
- Novo módulo `src/modules/webhooks`, seguindo a estrutura já usada nos demais módulos do OMS, e novo entry-point `src/worker.ts` ao lado de `src/server.ts` ([09:27]-[09:30] Bruno, Diego, Larissa).

Fora de escopo
- Alertar o cliente (por exemplo, por e-mail) quando o webhook dele acumula falhas consecutivas de entrega: adiado para uma fase futura, depois de medir o impacto real ([09:37]-[09:38] Larissa, Marcos).
- Rate limiting de envio para um mesmo cliente quando muitos pedidos mudam de status em um curto intervalo: a equipe decidiu observar se isso vira um problema real antes de implementar qualquer controle, em vez de resolver preventivamente nesta fase ([09:38]-[09:39] Diego, Larissa).
- Garantia de ordenação global de eventos entre pedidos diferentes caso o sistema escale para múltiplos workers em paralelo: tratada como limitação conhecida do modelo single-worker atual, a resolver apenas se e quando a escala exigir ([09:12]-[09:13] Diego, Bruno, Larissa).
- Dashboard visual para o cliente acompanhar os webhooks cadastrados: fora de escopo desta fase, projeto separado do time de frontend ([09:39]-[09:40] Larissa, Marcos).
- Estratégia de arquivamento das linhas já entregues na tabela de outbox: fora do escopo desta feature ([09:08] Diego).
- Webhooks inbound (o cliente enviar eventos para o sistema): a feature é exclusivamente outbound, o sistema só envia notificações, nunca recebe ([09:02]-[09:03] Marcos, Sofia).

---

### Requisitos funcionais

#### FR-001 Cadastro de webhook
Um usuário autenticado cadastra um endpoint de webhook para um customer, informando URL e a lista de status de interesse; a secret é gerada pela plataforma e devolvida apenas nesta resposta.

**Fluxo principal**
- Usuário autenticado envia `POST /api/v1/webhooks` com `customerId`, `url` e a lista de status observados.
- Sistema valida que a URL usa HTTPS.
- Sistema gera uma secret exclusiva para o endpoint.
- Sistema persiste o cadastro (URL, secret, customerId, status observados, estado ativo).
- Sistema retorna o cadastro criado, incluindo a secret gerada.

**Fluxos alternativos e exceções**
- URL cadastrada não usa HTTPS: requisição rejeitada antes de qualquer persistência.

**Erros previstos**
- `WEBHOOK_INVALID_URL` (400): URL não usa HTTPS.

**Prioridade:** Alta (hipótese: é o requisito de entrada; sem ele nenhum outro fluxo da feature existe)

---

#### FR-002 Edição de webhook
Um usuário autenticado atualiza a URL, a lista de status observados ou o estado ativo de um webhook já cadastrado.

**Fluxo principal**
- Usuário autenticado envia `PATCH /api/v1/webhooks/:id` com os campos a alterar.
- Sistema valida a existência do webhook e, se a URL for alterada, que ela usa HTTPS.
- Sistema persiste a alteração.
- Sistema retorna o webhook atualizado (a secret nunca é reexibida em edição).

**Fluxos alternativos e exceções**
- Webhook não existe ou não pertence ao customer informado.
- Nova URL não usa HTTPS.

**Erros previstos**
- `WEBHOOK_NOT_FOUND` (404).
- `WEBHOOK_INVALID_URL` (400).

**Prioridade:** Alta (hipótese: parte do CRUD básico decidido na reunião)

---

#### FR-003 Remoção de webhook
Um usuário autenticado remove um cadastro de webhook, interrompendo o recebimento de novas notificações naquele endpoint.

**Fluxo principal**
- Usuário autenticado envia `DELETE /api/v1/webhooks/:id`.
- Sistema remove o cadastro.
- Sistema retorna confirmação sem corpo de resposta.

**Fluxos alternativos e exceções**
- Webhook não existe.

**Erros previstos**
- `WEBHOOK_NOT_FOUND` (404).

**Prioridade:** Média (hipótese: completa o CRUD, mas não bloqueia o caminho principal de notificação)

---

#### FR-004 Listagem de webhooks de um customer
Um usuário autenticado consulta os webhooks cadastrados de um customer, de forma paginada.

**Fluxo principal**
- Usuário autenticado envia `GET /api/v1/webhooks` com `customerId` e parâmetros de paginação.
- Sistema retorna a lista paginada de webhooks do customer.

**Fluxos alternativos e exceções**
- Nenhuma exceção específica além do comportamento padrão de paginação já usado nos demais módulos.

**Erros previstos**
- Nenhum específico do módulo webhooks.

**Prioridade:** Média (hipótese)

---

#### FR-005 Histórico de entregas de um webhook
Um usuário autenticado consulta o histórico das últimas entregas de um webhook: sucesso ou falha, status HTTP, tempo de resposta.

**Fluxo principal**
- Usuário autenticado envia `GET /api/v1/webhooks/:id/deliveries` com paginação.
- Sistema retorna a lista paginada de tentativas de entrega, até os últimos 100 registros.

**Fluxos alternativos e exceções**
- Webhook não existe.

**Erros previstos**
- `WEBHOOK_NOT_FOUND` (404).

**Prioridade:** Média (hipótese: suporte operacional/diagnóstico, não bloqueia a entrega em si)

---

#### FR-006 Rotação de secret
Um usuário autenticado solicita a rotação da secret de um webhook; a secret antiga permanece válida por 24 horas em paralelo com a nova, para o cliente ter tempo de migrar.

**Fluxo principal**
- Usuário autenticado envia `POST /api/v1/webhooks/:id/rotate-secret`.
- Sistema gera uma nova secret para o endpoint.
- Sistema marca a secret antiga com expiração em 24 horas a partir da rotação.
- Sistema retorna a nova secret e a data de expiração da secret anterior.

**Fluxos alternativos e exceções**
- Webhook não existe.

**Erros previstos**
- `WEBHOOK_NOT_FOUND` (404).

**Prioridade:** Alta (decisão explícita de segurança da Sofia, motivada por incidente anterior de vazamento de secret, [09:21]-[09:22] Sofia, Diego)

---

#### FR-007 Registro do evento de notificação na mudança de status
Ao mudar o status de um pedido, o sistema registra, dentro da mesma transação, um evento de notificação para cada webhook ativo do customer cujo filtro de status inclui o novo status.

**Fluxo principal**
- `OrderService.changeStatus` abre a transação já existente e executa as validações e efeitos colaterais de hoje (existência do pedido, transição válida, ajuste de estoque, update em `orders`, insert em `order_status_history`).
- Dentro da mesma transação, o sistema chama uma função de publicação de evento passando o cliente de transação em uso.
- Essa função busca os webhooks ativos do customer do pedido cujo filtro de status inclui o novo status.
- Para cada webhook correspondente, insere uma linha na outbox com o payload já renderizado (snapshot do estado do pedido no momento da mudança de status).
- A transação é commitada: mudança de status e evento(s) de outbox tornam-se visíveis atomicamente.

**Fluxos alternativos e exceções**
- Nenhum webhook ativo do customer observa o novo status: nenhuma linha é inserida na outbox ([09:34] Bruno, Diego).
- Falha ao inserir o evento na outbox: toda a transação sofre rollback, não pode existir mudança de status sem o evento correspondente ([09:40]-[09:41] Bruno, Diego).

**Erros previstos**
- `WEBHOOK_SECRET_REQUIRED`: invariante interno, endpoint sem secret ativa válida.

**Prioridade:** Alta (é o ponto de integração que garante a atomicidade central da feature, decisão fechada nos ADRs)

---

#### FR-008 Processamento assíncrono e entrega HTTP assinada
Um processo worker separado busca eventos pendentes na outbox e realiza a entrega HTTP assinada ao endpoint cadastrado pelo cliente.

**Fluxo principal**
- O worker roda em loop de polling a cada 2 segundos.
- A cada iteração, busca um lote de eventos pendentes mais antigos (ordenados por data de criação).
- Para cada evento, monta o corpo já renderizado e os headers de assinatura (`X-Event-Id`, `X-Signature`, `X-Timestamp`, `X-Webhook-Id`).
- Realiza `POST` HTTP para a URL cadastrada, com timeout de 10 segundos.
- Em caso de resposta `2xx`, marca o evento como entregue e registra a tentativa no histórico de entregas.

**Fluxos alternativos e exceções**
- Timeout de 10 segundos sem resposta: tratado como falha de entrega.
- Resposta HTTP fora de `2xx`: tratado como falha de entrega.

**Erros previstos**
- Falha de entrega (timeout, erro de conexão ou status fora de `2xx`): registrada no histórico, segue para retry.

**Prioridade:** Alta (é o caminho principal de entrega da notificação ao cliente)

---

#### FR-009 Retry com backoff exponencial
Toda falha de entrega é reagendada seguindo uma progressão fixa de backoff, até um teto de tentativas.

**Fluxo principal**
- Progressão fixa de 5 tentativas: 1 minuto, 5 minutos, 30 minutos, 2 horas, 12 horas entre tentativas sucessivas ([09:15]-[09:17] Diego, Larissa).
- A cada falha, o contador de tentativas é incrementado e a próxima tentativa é agendada conforme a tabela de backoff.

**Fluxos alternativos e exceções**
- A 5ª tentativa falha: o evento segue para o fluxo de dead letter (FR-010).

**Erros previstos**
- Nenhum código de erro adicional; o resultado da falha é registrado no histórico de entregas.

**Prioridade:** Alta (garante a resiliência a indisponibilidade temporária do cliente, decisão explícita da equipe)

---

#### FR-010 Dead Letter Queue
Eventos que esgotam as tentativas de retry são movidos para uma tabela dead letter separada, preservando payload, motivo da falha e timestamp, para não perder nenhuma falha permanente silenciosamente.

**Fluxo principal**
- Após a 5ª tentativa falhar, o evento é removido da outbox.
- Uma linha correspondente é criada na tabela dead letter, com o payload completo, o motivo da última falha e o timestamp.

**Fluxos alternativos e exceções**
- Nenhuma além do fluxo de esgotamento de tentativas descrito em FR-009.

**Erros previstos**
- Nenhum específico; este requisito é, ele mesmo, o tratamento da falha permanente.

**Prioridade:** Alta (evita perda silenciosa de notificação, requisito explícito de Diego, [09:18])

---

#### FR-011 Replay administrativo de eventos em dead letter
Um usuário com role `ADMIN` reprocessa manualmente um evento em dead letter, recolocando-o na outbox como pendente.

**Fluxo principal**
- Usuário `ADMIN` envia `POST /api/v1/admin/webhooks/dead-letter/:id/replay`.
- Sistema valida que o usuário tem role `ADMIN`.
- Sistema reinsere o evento na outbox com status pendente e contador de tentativas reiniciado, disponível para o próximo ciclo do worker.
- A ação é registrada em log de auditoria, identificando o usuário `ADMIN` e o momento da ação ([09:36] Sofia).

**Fluxos alternativos e exceções**
- Usuário sem role `ADMIN` tenta executar o replay.
- Evento informado não existe na dead letter.

**Erros previstos**
- `401`/`403`: sem token ou sem role `ADMIN`.
- `WEBHOOK_DEAD_LETTER_NOT_FOUND` (hipótese, por analogia ao padrão `WEBHOOK_NOT_FOUND`; não citado literalmente na reunião).

**Prioridade:** Alta (é a única via de recuperação manual de uma falha permanente de entrega)

---

### Requisitos não funcionais

Performance
- Timeout de 10 segundos por chamada HTTP de entrega ([09:42] Diego).
- Ciclo de polling do worker de 2 segundos, definindo o piso de latência de entrega ([09:09]-[09:10] Diego, Larissa).
- Latência de entrega, no cenário sem falhas, dentro do teto de 10 segundos definido pelos clientes como "tempo real" ([09:02] Marcos).

Disponibilidade
- Meta de disponibilidade não foi discutida numericamente na reunião. Hipótese (default do processo): 99.9% para a API de configuração de webhook e para o worker, por se tratar de uma feature com clientes B2B externos dependendo de uma entrega confiável.

Segurança e autorização
- Assinatura HMAC-SHA256 do corpo de cada notificação, enviada no header `X-Signature` ([09:20] Sofia).
- Secret exclusiva por endpoint de webhook, nunca uma secret global da plataforma ([09:21] Sofia).
- Rotação de secret com grace period de 24 horas ([09:21] Sofia).
- URL do webhook obrigatoriamente HTTPS; cadastro com HTTP é recusado com erro de validação ([09:23] Sofia).
- Endpoint de replay administrativo restrito a usuários com role `ADMIN`, reaproveitando o middleware `requireRole` já existente ([09:36] Sofia).

Observabilidade
- Reaproveita o logger Pino já configurado no projeto, sem introduzir nova biblioteca ([09:29] Bruno).
- O mecanismo de redação de campos sensíveis do logger precisa ser estendido para não vazar secret e assinatura em log (hipótese fundamentada em código, `src/shared/logger/index.ts`, seguindo o mesmo padrão já usado para senha e token).
- Métricas específicas de outbox pendente, tentativas de entrega e tamanho da dead letter não foram citadas literalmente na reunião; ficam marcadas como hipótese, detalhadas no FDD.

Confiabilidade e integridade de dados
- A inserção do evento de webhook na outbox precisa ser transacional junto com a mudança de status: nunca pode existir mudança de status sem o evento correspondente, nem evento sem a mudança de status ter sido efetivamente commitada ([09:40]-[09:41] Bruno, Diego).
- Garantia de entrega at-least-once: o cliente pode receber o mesmo evento mais de uma vez e é responsável por deduplicar usando o `X-Event-Id` ([09:24]-[09:25] Diego).
- Nenhum payload de evento acima de 64KB é enviado; se ultrapassar o limite, a tentativa é rejeitada, nunca truncada ([09:23]-[09:24] Sofia, Diego, Larissa).

Compatibilidade e portabilidade
- O novo módulo é montado sob o mesmo prefixo de versão de API já usado pelos demais módulos do OMS, sem introduzir uma nova convenção de versionamento (hipótese, por consistência com o padrão observado no código).
- Nenhum contrato dos módulos já existentes (pedidos, clientes, produtos, usuários, autenticação) é alterado.

Compliance
- A ação de replay administrativo de dead letter é registrada em log de auditoria, identificando o usuário `ADMIN` que a executou e quando ([09:36] Sofia).

Acessibilidade
- Não aplicável a esta feature: trata-se de uma API B2B consumida por sistemas de clientes, sem interface visual voltada ao usuário final.

---

### Arquitetura e abordagem

Abordagem
- Padrão Outbox sobre o MySQL já existente: o evento de notificação é inserido na mesma transação SQL que já atualiza o pedido, garantindo atomicidade sem introduzir infraestrutura nova (ADR-001).
- Processamento assíncrono por um worker em processo separado, em polling, desacoplando a disponibilidade dos clientes externos da transação de mudança de status (ADR-002).

Componentes
- API existente do OMS, estendida com o novo módulo `src/modules/webhooks` (controller, service, repository, routes, schemas), seguindo a estrutura já usada nos demais módulos.
- Nova tabela de outbox de eventos de webhook, populada dentro da transação de mudança de status.
- Novo processo `src/worker.ts`, com sua própria instância de cliente de banco, responsável pelo polling, envio HTTP assinado e gestão de retry/dead letter.
- Nova tabela de dead letter para eventos que esgotaram as tentativas de entrega.

Integrações
- Extensão do método `changeStatus` do serviço de pedidos (`src/modules/orders/order.service.ts`) para publicar o evento de webhook dentro da mesma transação.
- Chamadas HTTP outbound do worker para as URLs cadastradas pelos clientes, autenticadas por HMAC-SHA256.

---

### Decisões e trade-offs

#### Decisão: Padrão Outbox no MySQL para entrega de eventos de webhook
- **Justificativa:** a transação de mudança de status já é pesada e não pode ficar acoplada à disponibilidade de um cliente externo; o outbox garante atomicidade entre a mudança de status e o registro do evento sem exigir infraestrutura nova (ADR-001).
- **Trade-off:** introduz uma tabela adicional que exige indexação própria e, no futuro, estratégia de arquivamento (fora do escopo desta feature).

#### Decisão: Worker em processo separado com polling de 2 segundos
- **Justificativa:** o worker precisa sobreviver a um reinício da API, e o MySQL não tem um mecanismo equivalente ao `LISTEN`/`NOTIFY` do Postgres para acionar processamento reativo (ADR-002).
- **Trade-off:** a latência mínima de entrega fica presa ao ciclo de polling de 2 segundos, e a ordenação de eventos só é garantida por pedido individual enquanto o processamento for single-worker.

#### Decisão: Retry com backoff exponencial (5 tentativas) e Dead Letter Queue em tabela separada
- **Justificativa:** cobre janelas reais de indisponibilidade de cliente (já houve caso de 2 horas por manutenção planejada) sem deixar eventos pendurados indefinidamente (ADR-003).
- **Trade-off:** exige um endpoint administrativo adicional, com auditoria, para reprocessamento manual dos eventos que esgotam as tentativas.

#### Decisão: Autenticação HMAC-SHA256 com secret por endpoint e rotação com grace period de 24h
- **Justificativa:** o cliente precisa validar autenticidade e integridade do payload recebido; uma secret por endpoint isola o impacto de um vazamento a um único cadastro, motivada por um incidente anterior de vazamento de secret em log de cliente (ADR-004).
- **Trade-off:** a plataforma passa a gerenciar o ciclo de vida completo de cada secret (geração, armazenamento seguro, rotação, expiração), em vez de um segredo único e estático.

#### Decisão: Garantia at-least-once com idempotência via `X-Event-Id`
- **Justificativa:** garantir exactly-once exigiria coordenação adicional entre os dois lados, desproporcional ao ganho; at-least-once com identificador de evento é o padrão de mercado (ADR-005).
- **Trade-off:** joga a responsabilidade de deduplicação para o lado do cliente.

#### Decisão: Reuso dos padrões arquiteturais já existentes no projeto
- **Justificativa:** o projeto já tem uma estrutura modular consolidada por domínio, classes de erro próprias, logger e middleware de autorização por papel; seguir o mesmo padrão mantém consistência arquitetural sem introduzir nova biblioteca ou convenção (ADR-006).
- **Trade-off:** limitações já existentes no error middleware e nos padrões atuais se propagam automaticamente para o novo módulo.

---

### Dependências

#### Organizacional: Revisão de segurança dedicada da Sofia antes do deploy
A entrega em produção depende de pelo menos dois dias úteis reservados para a Sofia (Engenharia de Segurança) revisar com calma o código de geração e verificação de HMAC e de geração/armazenamento de secret, antes do deploy ([09:46] Sofia). O cronograma de três sprints estimado por Larissa já contempla essa revisão no fim ([09:46] Larissa).

---

### Riscos e mitigação

#### Atraso na entrega gera risco de churn comercial da Atlas
- **Probabilidade:** média (prazo de três sprints é apertado e depende da revisão de segurança da Sofia antes do deploy)
- **Impacto:** alto (a Atlas sinalizou possível migração para concorrente em caso de atraso, [09:00] Marcos)
- **Mitigação:**
  - Revisão de segurança da Sofia já reservada, com pelo menos dois dias úteis dedicados antes do deploy ([09:46] Sofia)
  - Prazo de três sprints já inclui essa revisão no cronograma ([09:46] Larissa)
- **Plano de contingência:** acompanhamento periódico do progresso com Marcos para acionamento antecipado do cliente caso o cronograma derrape

#### Cliente externo indisponível além da janela de retry (~15 horas)
- **Probabilidade:** média (já houve caso real de indisponibilidade de 2 horas por manutenção planejada, [09:16] Diego)
- **Impacto:** médio (o evento não é perdido, mas exige reprocessamento manual via replay)
- **Mitigação:**
  - Retenção do evento em dead letter com payload completo e motivo da falha
  - Endpoint de replay manual disponível assim que o cliente normalizar ([09:18] Diego)
- **Plano de contingência:** nenhum reprocessamento automático além da janela de 15 horas; a intervenção manual via replay é o próprio plano de contingência

#### Revisão de segurança aponta mudanças estruturais tarde no cronograma
- **Probabilidade:** baixa (a revisão já está reservada com antecedência dentro do cronograma, [09:46] Sofia, Larissa)
- **Impacto:** alto se ocorrer (mudanças estruturais de última hora no código de HMAC/secret podem atrasar o deploy já comprometido com a Atlas)
- **Mitigação:**
  - Reserva antecipada de dois dias úteis de revisão dedicada, dentro do cronograma de três sprints, e não como etapa posterior a ele ([09:46]-[09:47] Sofia, Larissa)
- **Plano de contingência:** priorizar os ajustes de segurança apontados pela Sofia acima de qualquer outro item pendente do escopo, mesmo que isso signifique adiar itens de prioridade menor desta mesma entrega

#### Crescimento não controlado da tabela de outbox
- **Probabilidade:** alta no médio/longo prazo (a tabela acumula eventos entregues sem estratégia de arquivamento nesta fase, [09:08] Diego)
- **Impacto:** médio (pode degradar a performance de leitura do worker ao longo do tempo)
- **Mitigação:**
  - Índices em status e data de criação já previstos desde a decisão inicial (ADR-001)
- **Plano de contingência:** arquivamento ou purge de linhas entregues após período definido, fora do escopo desta feature, a ser revisado quando o volume justificar

#### Limitação de ordenação ao escalar para múltiplos workers
- **Probabilidade:** baixa no curto prazo (a arquitetura atual é single-worker por decisão explícita, ADR-002)
- **Impacto:** baixo hoje; potencialmente médio se a escala futura exigir múltiplos workers em paralelo
- **Mitigação:**
  - Documentar a limitação como conhecida e aceita para o escopo atual (ADR-002)
- **Plano de contingência:** particionamento por pedido ou lock pessimista, avaliado apenas se e quando a escala exigir ([09:13] Diego)

---

### Critérios de aceitação
Checklist objetivo que define se a feature está pronta.

- Um usuário autenticado consegue cadastrar, editar, remover e listar webhooks de um customer pela API.
- Toda mudança de status para a qual existe pelo menos um webhook ativo correspondente gera um evento na outbox dentro da mesma transação; se a transação sofrer rollback, o evento não existe.
- Notificações são entregues com latência mínima de 2 segundos e, no cenário sem falhas, dentro do teto de 10 segundos combinado com os clientes.
- Toda notificação enviada carrega uma assinatura HMAC-SHA256 válida e um `X-Event-Id` único e estável entre tentativas do mesmo evento.
- Uma falha de entrega é retentada na progressão de backoff de 1 minuto, 5 minutos, 30 minutos, 2 horas e 12 horas, antes de ser movida para dead letter.
- Um usuário com role `ADMIN` consegue reprocessar manualmente um evento em dead letter, e essa ação fica registrada em auditoria; qualquer outra role recebe erro de permissão.
- Após rotação de secret, a secret anterior continua válida por exatamente 24 horas em paralelo com a nova.
- Nenhum payload de evento acima de 64KB é enviado ao cliente; a tentativa é rejeitada antes do envio, sem truncamento.
- Um usuário autenticado consegue consultar o histórico das últimas entregas (sucesso ou falha) de um webhook cadastrado.
- Webhooks cadastrados ou editados com URL que não seja HTTPS são rejeitados antes de qualquer persistência.

---

### Testes e validação

Tipos de teste obrigatórios
- Testes unitários para a lógica crítica: cálculo de backoff exponencial, geração e verificação de HMAC-SHA256, validação de URL HTTPS, e filtro de status aplicado na inserção da outbox (hipótese: a reunião não detalha os tipos de teste, mas trata essas regras como críticas).
- Testes de integração cobrindo a extensão do `changeStatus`, garantindo que o rollback da transação elimina o evento de outbox junto com a mudança de status ([09:46] Larissa: "Integração no order.service e testes ponta a ponta").
- Revisão de segurança dedicada, feita pela Sofia, sobre o código de HMAC e de geração/armazenamento de secret, como gate obrigatório antes do deploy ([09:46]-[09:47] Sofia, Larissa).

Estratégia de validação
- Testes automatizados (unitários e de integração) cobrindo o fluxo ponta a ponta mencionado explicitamente no planejamento de sprints da reunião ([09:46] Larissa).
- Revisão de segurança dedicada da Sofia, com pelo menos dois dias úteis reservados, como validação obrigatória antes de qualquer deploy em produção ([09:46] Sofia).
