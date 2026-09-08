# ADR-004: Autenticação HMAC-SHA256 com secret por endpoint e rotação com grace period

**Status:** Aceito
**Decisores:** Sofia (Engenharia de Segurança), Diego (Engenheiro Sênior, Plataforma)

---

## Contexto

Os eventos de webhook carregam dados de pedidos e saem da infraestrutura da empresa em direção a sistemas de terceiros. O cliente que recebe o evento precisa conseguir validar que a requisição veio realmente da empresa e que o payload não foi adulterado no caminho. A equipe já teve um incidente em que a secret de um cliente vazou pelo log de aplicação dele, o que reforça a necessidade de limitar o raio de impacto de um vazamento de secret.

## Decisão

Assinar o corpo de cada requisição de webhook com HMAC-SHA256, enviando a assinatura no header `X-Signature` para o cliente verificar do lado dele. Cada endpoint de webhook cadastrado por um cliente tem sua própria secret, gerada pela plataforma no momento do cadastro, e não uma secret global compartilhada entre todos os endpoints. A secret é rotacionável via endpoint da API: ao rotacionar, a secret antiga continua válida por 24 horas em paralelo com a nova, dando tempo para o cliente migrar seus sistemas antes que a secret antiga expire.

## Alternativas Consideradas

### Secret global da plataforma, compartilhada entre todos os endpoints
Usar uma única secret para assinar todos os eventos de todos os clientes. Descartada porque, se essa secret única vazasse, todos os clientes e todos os endpoints ficariam comprometidos simultaneamente; uma secret por endpoint isola o impacto de um vazamento a um único cadastro.

## Consequências

**Positivas**
- O cliente consegue verificar autenticidade e integridade de cada evento recebido usando um algoritmo padrão de mercado (HMAC-SHA256), amplamente suportado por bibliotecas já existentes.
- O vazamento da secret de um endpoint fica isolado a esse endpoint específico, sem comprometer os demais clientes ou cadastros.
- A rotação com grace period de 24 horas permite trocar a secret sem downtime de validação para o cliente.

**Negativas**
- Exige que a plataforma gerencie o ciclo de vida completo de cada secret (geração, armazenamento seguro, rotação, expiração da secret antiga), em vez de um segredo único e estático.
- A tabela de configuração de webhook passa a armazenar um dado sensível (a secret) além de url, customer_id e estado ativo, exigindo cuidado adicional de proteção desse campo.

## Referências

- TRANSCRICAO.md, trecho [09:19] a [09:22]
