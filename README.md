# Da Reunião ao Documento: Design Docs Gerados por IA

## Sobre esta abordagem

A estratégia adotada neste projeto foi usar o desafio como oportunidade para produzir ferramentas reutilizáveis para a vida real, não apenas para gerar os documentos deste desafio específico. Por esse motivo, optei por criar uma skill de IA dedicada a cada artefato do pacote de design docs (PRD, RFC, FDD, ADRs e Tracker), em vez de escrever prompts avulsos e descartáveis para cada documento. O enunciado original do desafio está preservado, sem alterações, em [`descricao-desafio.md`](descricao-desafio.md).

## ADR

As ADRs foram geradas pela skill `criador-adr`, que primeiro lê todo material de entrada disponível (a transcrição da reunião e o código-fonte da aplicação) e só depois entrevista quem está usando a skill sobre o que restou em aberto. A skill aplicou um critério de elegibilidade (estrutural, evidente e estável) para separar, dentro da transcrição, o que realmente merece virar uma decisão registrada do que é só detalhe de implementação ou não foi decidido. Isso gerou uma lista de decisões candidatas, apresentada para confirmação antes de qualquer arquivo ser escrito; nessa etapa uma candidata de baixa confiança (o momento em que o payload do evento é montado) foi descartada a pedido, ficando só as 6 decisões centrais da reunião. Para cada uma delas, a skill extraiu contexto, decisão, alternativas descartadas e consequências direto da transcrição e do código (padrão de erros, middleware de autorização, logger, estrutura de módulos), sem precisar de nenhuma pergunta adicional, já que o material cobria todos os campos exigidos pelo formato MADR. O resultado foi revisado criticamente numa segunda passada: 5 das citações do Tracker apontavam para a fala errada da transcrição (próxima do assunto, mas não a fala que de fato fecha a decisão) e foram corrigidas para a citação literal correspondente.

Comando usado para gerar as ADRs:

```
/criador-adr @repo-base/descricao-desafio.md Vamos criar as ADRs com base no desafio. Use a skill para fazer essa atividade atendendo rigorosamente ao que o desafio pede. Os materiais de entrada são @repo-base/TRANSCRICAO.md e @repo-base/src/.
```

## RFC

O `docs/RFC.md` foi gerado pela skill `criador-rfc`, usando como entrada a transcrição da reunião (`TRANSCRICAO.md`), o código-fonte da aplicação e as 6 ADRs já fechadas nesta etapa do processo. O desafio exige que a documentação seja produzida depois de a decisão já ter sido tomada na reunião, o inverso do fluxo usual em que a RFC abre a discussão e as ADRs vêm depois. Deixei essa inversão explícita no comando, apontando que o raciocínio continua o mesmo (proposta, alternativas, questões em aberto), só a ordem de escrita dos documentos que mudou para se adequar ao material disponível. A skill extraiu do material o contexto do problema, a proposta técnica de alto nível e as alternativas descartadas na reunião, referenciando as ADRs já existentes em vez de repetir o detalhamento de cada decisão, e não precisou de nenhuma pergunta adicional para fechar o documento, já que a transcrição e as ADRs cobriam integralmente os campos exigidos pelo formato. O resultado saiu consistente com o restante do pacote já na primeira geração, sem correções necessárias na revisão.

Comando usado para gerar o RFC:

```
/criador-rfc  @README.md @repo-base/TRANSCRICAO.md @repo-base/src/ use o criador de RFC para criar uma nova RFC para o projeto. Normalmente uma RFC é criada para discutir um assunto para tomar desições basendo-se nas discussões e posteriomente gerar a ADR a partir dela, mas a RFC está sendo criada depois porque a discussão ocorreu em uma reunião, que é a transcrição fornecida. O fluxo em si não mudou, mas a escrita dos documentos estão em ordem mais apropriada para melhorar o resultado do processo. Neste caso, tenha em vista as ADRs que já foram criadas em @repo-base/docs/adrs.
```

## FDD

A skill `criador-fdd` gerou o `docs/FDD.md` depois das ADRs e do RFC, na ordem alterada deste desafio, usando a transcrição, o RFC e as 6 ADRs fechadas como entrada, sem tratar o PRD (ainda placeholder) como fonte. Explorou o código-fonte (changeStatus, classes de erro, middlewares, logger, rotas, schema Prisma) para a seção "Integração com o sistema existente". Como o conteúdo técnico já estava fechado, dispensei entrevista longa: os poucos pontos sem origem real ficaram marcados como hipótese no próprio documento. A autorrevisão final confirmou os critérios de aceite, com ressalva sobre payload JSON incompleto em 2 dos 8 contratos.

Comando usado para gerar o FDD:

```
/criador-adr @repo-base/descricao-desafio.md Use a skill FDD para gerar a FDD seguindo rigorosamente as regras do desafio. Perceba que a sequência de criação dos documentos foram mudados; considere no contexto apenas as arquivos ADR e RFC.
```
