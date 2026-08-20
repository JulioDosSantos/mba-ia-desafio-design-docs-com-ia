# ADR-001: Outbox no MySQL na transação do pedido

## Status

Aceito para o design. A implementação da funcionalidade não faz parte desta entrega.

## Contexto

Uma mudança de status de pedido já atualiza o pedido, registra o histórico e pode ajustar o estoque dentro de uma transação. A reunião rejeitou fazer uma chamada HTTP síncrona nesse caminho: um consumidor lento ou indisponível bloquearia outros pedidos ou obrigaria a tratar a mudança do pedido como falha. Isso foi discutido por Larissa, Bruno e Diego entre 09:03 e 09:06.

O objetivo é preservar o evento quando a transação principal for confirmada e descartá-lo quando ela sofrer rollback. O MySQL existente foi escolhido para evitar infraestrutura adicional. A filtragem por status também deve acontecer na inserção: se nenhum endpoint do cliente tiver interesse no status, nenhuma linha de evento deve ser criada. Essas decisões foram confirmadas entre 09:06 e 09:08 e entre 09:31 e 09:34, por Diego, Larissa, Marcos e Bruno.

## Decisão

Registrar o evento outbound em uma Outbox no MySQL dentro da mesma transação que altera o pedido e seu histórico. A integração recebe o transaction client da transação atual; se o registro da Outbox falhar, a mudança do pedido também falha. O evento deve receber seu UUID e guardar o payload como snapshot no momento da inserção, para que uma alteração posterior do pedido não mude o conteúdo que será entregue.

O nome final do modelo/tabela e seus campos detalhados ficam para o FDD. A decisão aqui é sobre a atomicidade e o uso do MySQL existente, não sobre um schema já implementado.

## Alternativas Consideradas

| Alternativa | Motivo da rejeição | Trade-off |
|---|---|---|
| HTTP síncrono dentro da transação | Um endpoint lento ou fora do ar travaria a mudança de status ou faria o pedido sofrer rollback. | Seria simples de chamar, mas acoplaria a disponibilidade externa à transação crítica do pedido. |
| Redis Streams ou outra fila externa | Exigiria subir e operar infraestrutura adicional; Diego considerou overengineering para o time e o cenário. | Poderia oferecer recursos de consumo dedicados, mas aumentaria dependências e complexidade operacional. |

## Consequências

### Positivas

- A confirmação do pedido e o registro do evento têm a mesma atomicidade.
- O envio HTTP deixa de bloquear a transação de mudança de status.
- O desenho usa o MySQL e o Prisma já existentes, sem introduzir Redis nesta entrega.
- O filtro na inserção reduz eventos persistidos quando nenhum endpoint tem interesse.
- O snapshot preserva o estado correto do pedido no momento da transição.

### Negativas

- A Outbox passa a compartilhar capacidade e manutenção com o banco principal.
- O envio deixa de ser síncrono e depende do worker e de polling, portanto há latência eventual.
- Será necessário definir retenção, estados e índices da Outbox no detalhamento técnico; o arquivamento após aproximadamente 30 dias está fora do escopo atual.
- A garantia de atomicidade não impede duplicatas na entrega; isso é tratado no [ADR-005](ADR-005-at-least-once-e-x-event-id.md).

## Relações e referências

- O consumo, polling e a limitação de ordenação estão no [ADR-002](ADR-002-worker-separado-e-polling-de-2s.md).
- Retry e DLQ estão no [ADR-003](ADR-003-retry-com-backoff-e-dlq.md).
- A integração com `OrderService.changeStatus` e os padrões do código estão no [ADR-006](ADR-006-reuso-dos-padroes-do-projeto.md).
- A proposta de alto nível será consolidada no [RFC](../RFC.md) e o detalhamento implementável no [FDD](../FDD.md).
- Evidências primárias: [TRANSCRICAO.md](../../TRANSCRICAO.md), `[09:03–09:08]` e `[09:31–09:34]`.
