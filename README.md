# Da Reunião ao Documento: Design Docs Gerados por IA

## Sobre o desafio

Este desafio pede para transformar a transcrição de uma reunião técnica (`TRANSCRICAO.md`) e o código-fonte já existente de um Order Management System em um pacote completo de design docs (PRD, RFC, FDD, ADRs e Tracker) para uma nova feature de Sistema de Webhooks de Notificação de Pedidos, usando IA como ferramenta principal de produção. A entrega é puramente documental: o código não pode ser alterado, e toda informação registrada nos documentos precisa ser rastreável à transcrição ou ao código, nunca inventada.

A estratégia que adotei foi usar o desafio como oportunidade para produzir ferramentas reutilizáveis para a vida real, não apenas para gerar os documentos deste desafio específico. Por esse motivo, optei por criar uma skill de IA dedicada a cada artefato do pacote (PRD, RFC, FDD, ADRs e Tracker), em vez de escrever prompts avulsos e descartáveis para cada documento. O enunciado original do desafio está preservado, sem alterações, em [`descricao-desafio.md`](descricao-desafio.md).

## Ferramentas de IA utilizadas

- **Claude Code CLI**, com quatro skills próprias (`criador-adr`, `criador-rfc`, `criador-fdd`, `criador-prd`): ferramenta principal de produção. Leu a transcrição e o código-fonte, estruturou e gerou o conteúdo de cada documento, e entrevistou sobre o que restava em aberto quando o material de entrada não cobria algum campo exigido.
- **Perplexity.ai** (via navegador): usado para validar os arquivos de definição das quatro skills antes de aplicá-las neste desafio. O resultado trouxe um alerta de que as skills eram genéricas demais para o desafio específico. Desconsiderei o aviso, porque a genericidade era proposital: o objetivo da estratégia era construir ferramentas reaproveitáveis em outros projetos, não prompts feitos sob medida só para este desafio.

## Workflow adotado

Ordem de produção, invertida em relação ao fluxo usual, seguindo a sequência didática pedida pelo desafio:

1. **ADRs**: as 6 decisões centrais da reunião viraram o esqueleto do pacote, produzidas antes de qualquer outro documento.
2. **RFC**: consolidou a proposta técnica em cima das ADRs já fechadas.
3. **FDD**: detalhou a implementação com base no RFC e nas ADRs, sem tratar o PRD (ainda placeholder) como fonte.
4. **PRD**: por último, como consolidação de tudo que já estava fechado nos documentos anteriores.
5. **Tracker**: alimentado em paralelo, documento a documento, reaproveitando as citações já levantadas durante a extração de cada um.
6. **README**: escrito por último, documentando o processo já concluído.

Cada skill foi invocada apontando explicitamente para os materiais de entrada relevantes (transcrição, código-fonte e os documentos já fechados nas etapas anteriores) via referências de arquivo (`@repo-base/...`) no próprio prompt, para reduzir a chance de a IA inventar contexto que não estava disponível.

## Prompts customizados

Comando usado para gerar as ADRs:

```
/criador-adr @repo-base/descricao-desafio.md Vamos criar as ADRs com base no desafio. Use a skill para fazer essa atividade atendendo rigorosamente ao que o desafio pede. Os materiais de entrada são @repo-base/TRANSCRICAO.md e @repo-base/src/.
```

Comando usado para gerar o RFC:

```
/criador-rfc  @README.md @repo-base/TRANSCRICAO.md @repo-base/src/ use o criador de RFC para criar uma nova RFC para o projeto. Normalmente uma RFC é criada para discutir um assunto para tomar desições basendo-se nas discussões e posteriomente gerar a ADR a partir dela, mas a RFC está sendo criada depois porque a discussão ocorreu em uma reunião, que é a transcrição fornecida. O fluxo em si não mudou, mas a escrita dos documentos estão em ordem mais apropriada para melhorar o resultado do processo. Neste caso, tenha em vista as ADRs que já foram criadas em @repo-base/docs/adrs.
```

Comando usado para gerar o FDD:

```
/criador-adr @repo-base/descricao-desafio.md Use a skill FDD para gerar a FDD seguindo rigorosamente as regras do desafio. Perceba que a sequência de criação dos documentos foram mudados; considere no contexto apenas as arquivos ADR e RFC.
```

Comando usado para gerar o PRD:

```
/criador-prd crie um PRD seguindo rigorosamente às especificações do @repo-base/descricao-desafio.md. Note que, embora o PRD normalmente seja o primeiro a ser criado, como o escopo é didático de um desafio, optou-se por deixá-lo por último. Isso significa que deve levar em conta os outros arquivos já criados como TRACKER, ADRs, RFC e FDD que estão em @repo-base/docs/. Como entrada, use @repo-base/TRANSCRICAO.md e @repo-base/src/.
```

## Iterações e ajustes

O processo levou 6 ciclos principais de geração e revisão crítica: a validação inicial das skills, um ciclo por documento e a correção final do README.

1. **Validação das skills no Perplexity.ai**: antes de usá-las neste desafio, colei os arquivos de definição das quatro skills no Perplexity.ai, via navegador, para uma segunda opinião. O resultado trouxe um alerta de que as skills eram genéricas demais para o desafio específico. Desconsiderei o aviso: a genericidade era o objetivo declarado da estratégia (ferramentas reutilizáveis, não prompts descartáveis para este desafio), então o "problema" apontado era, na verdade, a decisão de design.
2. **ADRs**: a skill apresentou uma lista de decisões candidatas antes de escrever qualquer arquivo; uma candidata de baixa confiança (o momento em que o payload do evento é montado) foi descartada a pedido, ficando só as 6 decisões centrais. Na revisão crítica da segunda passada, 5 das citações do Tracker apontavam para a fala errada da transcrição (próxima do assunto, mas não a fala que de fato fecha a decisão) e foram corrigidas para a citação literal correspondente.
3. **RFC**: saiu consistente com o restante do pacote já na primeira geração, sem correções necessárias na revisão.
4. **FDD**: a autorrevisão final identificou uma ressalva sobre payload JSON incompleto em 2 dos 8 contratos documentados (os dois endpoints sem corpo de requisição, DELETE e GET), registrada explicitamente no documento em vez de ser escondida.
5. **PRD**: ao tentar preencher, por entrevista, a ausência de números de impacto do problema (custo ou tempo perdido pelo polling), o processo foi interrompido: o formato de Tracker deste desafio só aceita `TRANSCRICAO` ou `CODIGO` como fonte, então uma resposta minha na conversa não teria como ser rastreada. Isso mudou o approach: todo ponto sem origem no material passou a ser assumido como hipótese, marcado explicitamente no texto, em vez de preenchido por entrevista.
6. **README fora do padrão exigido, despercebido até o fim**: o primeiro README gerado documentava o processo por skill usada (Sobre esta abordagem, ADR, RFC, FDD, PRD), mas não seguia a estrutura de seções obrigatória do desafio (Sobre o desafio, Ferramentas de IA utilizadas, Workflow adotado, Prompts customizados, Iterações e ajustes, Como navegar a entrega). Esse desvio passou despercebido por mim durante toda a produção dos demais documentos; só percebi ao chegar no fim e comparar o README com o padrão indicado no enunciado. Em nenhum momento anterior a IA alertou, por conta própria, que o README estava fora do formato exigido, mesmo tendo acesso ao enunciado completo; a correção só aconteceu depois de eu pedir explicitamente uma checagem do pacote inteiro contra a checklist de critérios de aceite.

## Como navegar a entrega

Ordem sugerida de leitura:

1. [`descricao-desafio.md`](descricao-desafio.md): enunciado original do desafio, preservado sem alterações.
2. [`docs/adrs/`](docs/adrs/): as 6 ADRs das decisões centrais da reunião, mais o índice em [`docs/adrs/README.md`](docs/adrs/README.md).
3. [`docs/RFC.md`](docs/RFC.md): proposta técnica consolidada, com links para as ADRs.
4. [`docs/FDD.md`](docs/FDD.md): especificação de implementação, incluindo a seção "Integração com o sistema existente".
5. [`docs/PRD.md`](docs/PRD.md): visão de produto, consolidando o que já estava fechado nos documentos anteriores.
6. [`docs/TRACKER.md`](docs/TRACKER.md): rastreabilidade de cada item à transcrição ou ao código.
