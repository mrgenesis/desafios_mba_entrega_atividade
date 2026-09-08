# ADR-005: Garantia at-least-once com idempotência via X-Event-Id

**Status:** Aceito
**Decisores:** Diego (Engenheiro Sênior, Plataforma), Sofia (Engenharia de Segurança), Marcos (Product Manager)
**Relacionado a:** ADR-003

---

## Contexto

Dada a combinação de retries com backoff (ver ADR-003) e um worker que processa eventos de forma assíncrona (ver ADR-002), existe a possibilidade real de o mesmo evento ser entregue mais de uma vez ao cliente, por exemplo se uma tentativa de entrega foi bem-sucedida do lado do cliente mas a confirmação não chegou de volta ao worker a tempo. Garantir entrega exactly-once (exatamente uma vez) exigiria coordenação adicional entre os dois lados (empresa e cliente), o que aumenta significativamente a complexidade da solução.

## Decisão

Garantir apenas entrega at-least-once (pelo menos uma vez): o cliente pode, em alguns casos, receber o mesmo evento mais de uma vez, e deve estar preparado para isso. Para viabilizar a deduplicação do lado do cliente, cada evento carrega um `X-Event-Id` no header, contendo um UUID gerado no momento em que o evento entra na outbox e único por evento. O cliente é responsável por usar esse identificador para dedicar eventualmente recebidos em duplicidade.

## Alternativas Consideradas

### Garantia exactly-once
Garantir que cada evento seja entregue exatamente uma vez, sem duplicações possíveis. Descartada porque exigiria coordenação entre os dois lados (confirmação transacional de recebimento, por exemplo), aumentando muito a complexidade da solução para um ganho que a equipe considerou não justificar o esforço, frente ao padrão de mercado já validado por outras plataformas (citadas Stripe e GitHub) usando at-least-once com identificador de evento.

## Consequências

**Positivas**
- Solução simples de implementar e operar, seguindo um padrão de mercado já validado por outras plataformas de webhook (Stripe, GitHub).
- Resolve a grande maioria dos casos práticos de entrega sem exigir infraestrutura de coordenação adicional entre os dois lados.

**Negativas**
- Transfere para o cliente a responsabilidade de implementar a deduplicação usando o `X-Event-Id`, o que exige documentação clara no portal do desenvolvedor para que os clientes façam essa implementação corretamente.

## Referências

- TRANSCRICAO.md, trecho [09:24] a [09:26]
