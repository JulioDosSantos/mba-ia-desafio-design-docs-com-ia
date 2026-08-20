# ADR-005: At-least-once e `X-Event-Id`

## Status

Aceito para o design. A implementação da funcionalidade não faz parte desta entrega.

## Contexto

Entregas HTTP com retry podem ser repetidas: o produtor pode não saber se o consumidor recebeu a requisição antes de um timeout ou de uma falha de conexão. Diego explicou às 09:24–09:25 que o mesmo evento pode chegar duas vezes e que o consumidor deve estar preparado.

A equipe não quer impor coordenação complexa entre a plataforma e cada consumidor para prometer exactly-once. O evento precisa, portanto, possuir um identificador estável desde a entrada na Outbox. Esse identificador será enviado no payload como `event_id` e no header `X-Event-Id`; o snapshot do payload e a geração na inserção foram confirmados às 09:51–09:52.

## Decisão

Adotar semântica de entrega at-least-once. Gerar um UUID único quando o evento entrar na Outbox e propagar o mesmo valor como `event_id` no JSON e `X-Event-Id` no request. A documentação para consumidores deve exigir deduplicação por esse identificador.

Essa decisão não promete ordenação global nem impede que uma tentativa repetida seja observada pelo consumidor. A assinatura e o timestamp do request seguem o [ADR-004](ADR-004-hmac-sha256-e-segredo-por-endpoint.md); o retry que pode causar a duplicata segue o [ADR-003](ADR-003-retry-com-backoff-e-dlq.md).

## Alternativas Consideradas

| Alternativa | Motivo da rejeição | Trade-off |
|---|---|---|
| Exactly-once | Exigiria coordenação adicional dos dois lados e foi considerada complexidade desnecessária para o cenário. | Poderia simplificar a expectativa do consumidor, mas aumentaria complexidade e não seria garantível apenas pelo produtor HTTP. |
| Não enviar identificador de evento | Impediria o consumidor de reconhecer com segurança duas entregas do mesmo evento. | Reduziria um campo/header, mas transferiria a deduplicação para heurísticas frágeis. |

## Consequências

### Positivas

- O produtor pode fazer retry sem depender de uma confirmação transacional do sistema externo.
- O consumidor recebe uma chave explícita para deduplicação.
- O contrato permanece compatível com a tabela de entregas e com a DLQ, pois o evento mantém o mesmo identificador.
- A decisão evita prometer uma garantia que o transporte HTTP não consegue assegurar sozinho.

### Negativas

- A responsabilidade de idempotência fica com cada consumidor.
- Duplicatas continuam possíveis e precisam aparecer na documentação de integração.
- O identificador precisa permanecer estável no snapshot e em todos os retries; gerar outro UUID por tentativa quebraria a decisão.
- At-least-once não resolve a limitação de ordenação futura descrita no [ADR-002](ADR-002-worker-separado-e-polling-de-2s.md).

## Relações e referências

- A criação atômica e o snapshot na Outbox estão no [ADR-001](ADR-001-outbox-no-mysql.md).
- Retry e replay estão no [ADR-003](ADR-003-retry-com-backoff-e-dlq.md).
- HMAC e headers de segurança estão no [ADR-004](ADR-004-hmac-sha256-e-segredo-por-endpoint.md).
- A decisão será resumida no [RFC](../RFC.md) e formalizada nos contratos do [FDD](../FDD.md).
- Evidências primárias: [TRANSCRICAO.md](../../TRANSCRICAO.md), `[09:24–09:26]` e `[09:51–09:52]`.
