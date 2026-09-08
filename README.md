# Da Reunião ao Documento: Design Docs Gerados por IA

## Sobre esta abordagem

A estratégia adotada neste projeto foi usar o desafio como oportunidade para produzir ferramentas reutilizáveis para a vida real, não apenas para gerar os documentos deste desafio específico. Por esse motivo, optei por criar uma skill de IA dedicada a cada artefato do pacote de design docs (PRD, RFC, FDD, ADRs e Tracker), em vez de escrever prompts avulsos e descartáveis para cada documento. O enunciado original do desafio está preservado, sem alterações, em [`descricao-desafio.md`](descricao-desafio.md).

## ADR

As ADRs foram geradas pela skill `criador-adr`, que primeiro lê todo material de entrada disponível (a transcrição da reunião e o código-fonte da aplicação) e só depois entrevista quem está usando a skill sobre o que restou em aberto. A skill aplicou um critério de elegibilidade (estrutural, evidente e estável) para separar, dentro da transcrição, o que realmente merece virar uma decisão registrada do que é só detalhe de implementação ou não foi decidido. Isso gerou uma lista de decisões candidatas, apresentada para confirmação antes de qualquer arquivo ser escrito; nessa etapa uma candidata de baixa confiança (o momento em que o payload do evento é montado) foi descartada a pedido, ficando só as 6 decisões centrais da reunião. Para cada uma delas, a skill extraiu contexto, decisão, alternativas descartadas e consequências direto da transcrição e do código (padrão de erros, middleware de autorização, logger, estrutura de módulos), sem precisar de nenhuma pergunta adicional, já que o material cobria todos os campos exigidos pelo formato MADR. O resultado foi revisado criticamente numa segunda passada: 5 das citações do Tracker apontavam para a fala errada da transcrição (próxima do assunto, mas não a fala que de fato fecha a decisão) e foram corrigidas para a citação literal correspondente.

Comando usado para gerar as ADRs:

```
/criador-adr @repo-base/descricao-desafio.md Vamos criar as ADRs com base no desafio. Use a skill para fazer essa atividade atendendo rigorosamente ao que o desafio pede. Os materiais de entrada são @repo-base/TRANSCRICAO.md e @repo-base/src/.
```
