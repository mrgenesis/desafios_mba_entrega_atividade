# ADR-002: Worker em processo separado com polling de 2 segundos

**Status:** Aceito
**Decisores:** Larissa (Tech Lead), Diego (Engenheiro Sênior, Plataforma), Bruno (Engenheiro Pleno, Pedidos)
**Relacionado a:** ADR-001

---

## Contexto

Os clientes B2B que pediram a feature (Atlas Comercial, MaxDistribuição e Nova Cargo) consideram "tempo real" qualquer notificação entregue em menos de 10 segundos. Os eventos de notificação ficam registrados na tabela `webhook_outbox` (ver ADR-001) e precisam ser lidos e entregues por algum processo. Esse processo não pode viver dentro da mesma instância da API: se a API reiniciar (deploy, crash, etc.), o processamento da outbox não pode ser interrompido junto.

## Decisão

Criar um processo Node separado, como um novo entry-point do projeto (`src/worker.ts`, executado via `npm run worker`, seguindo o mesmo padrão de `src/server.ts`), que se conecta ao mesmo banco de dados com sua própria instância de `PrismaClient` (mesma `DATABASE_URL`, mas instância nova porque `PrismaClient` é por processo). Esse worker faz polling em loop, a cada 2 segundos, buscando os eventos pendentes mais antigos na outbox, processando-os e marcando-os como entregues. Um único worker rodando processa os eventos em ordem de `created_at`, o que garante ordenação por pedido individual enquanto houver apenas uma instância do worker.

## Alternativas Consideradas

### Trigger de banco de dados para notificar o worker
Usar um trigger do MySQL para avisar o worker assim que um evento é inserido na outbox, tornando o processamento mais reativo do que o polling. Descartada porque o MySQL não tem um mecanismo nativo equivalente ao `LISTEN`/`NOTIFY` do Postgres: um trigger só executa SQL, não notifica um processo externo. Para contornar isso seria necessário improvisar algo como escrever em arquivo ou chamar um endpoint a partir do próprio banco, o que foi considerado uma solução estranha e frágil.

### Worker rodando na mesma instância de processo da API
Executar a lógica de processamento da outbox dentro do mesmo processo Node que serve a API HTTP. Descartada porque um reinício da API (deploy, crash, escalonamento) mataria o processamento da outbox junto, quebrando a garantia de entrega.

## Consequências

**Positivas**
- Atende com folga o requisito de latência dos clientes (abaixo de 10 segundos), já que o polling de 2 segundos implica uma latência mínima de 2 segundos no pior caso.
- O processamento de eventos fica resiliente a reinícios da API, por rodar em processo independente.

**Negativas**
- Não há garantia de ordenação global de eventos entre pedidos diferentes se, no futuro, o sistema escalar para múltiplos workers em paralelo; a garantia de ordem por pedido (`order_id`) só vale enquanto houver um único worker processando a outbox em ordem de `created_at`. Essa é uma limitação conhecida e aceita para o escopo atual.

## Referências

- `src/server.ts` (padrão de entry-point a ser replicado em `src/worker.ts`)
- TRANSCRICAO.md, trecho [09:09] a [09:14]
