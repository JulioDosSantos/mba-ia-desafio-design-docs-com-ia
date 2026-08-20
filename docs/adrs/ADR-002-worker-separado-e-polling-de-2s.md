# ADR-002: Worker separado e polling de 2 segundos

## Status

Aceito para o design. A implementação da funcionalidade não faz parte desta entrega.

## Contexto

A Outbox precisa de um processo que leia eventos pendentes e faça as chamadas HTTP. O MySQL não oferece um listener nativo para acordar esse processo, e um trigger executaria SQL, mas não notificaria adequadamente um processo externo. Diego e Bruno discutiram essa limitação às 09:09.

A meta de negócio aceita é receber a mudança de status em menos de 10 segundos. Por isso, Diego propôs polling a cada 2 segundos e Marcos confirmou que o intervalo atende; Larissa registrou a decisão às 09:09–09:10. O worker também não deve viver dentro da mesma instância da API: se a API reiniciar, o processamento não pode desaparecer com ela. O worker usa o mesmo banco e a mesma stack/Prisma, mas cada processo tem sua própria instância de `PrismaClient`, conforme a conversa de 09:11 e 09:29–09:30.

## Decisão

Executar o worker como processo separado da API, consultando a Outbox em polling a cada 2 segundos. O processamento deve buscar os eventos pendentes mais antigos em lote pequeno, ordená-los por `created_at` e marcar o resultado conforme o fluxo de entrega definido no [ADR-003](ADR-003-retry-com-backoff-e-dlq.md).

O cenário inicial é single-worker. Nesse cenário, a ordem por `created_at` oferece ordenação implícita por pedido, mas não constitui garantia de ordenação global caso haja paralelismo futuro. Particionamento por `order_id` e lock pessimista ficam postergados.

A reunião sugeriu um novo entrypoint e um script dedicado para o worker. A proposta R-001 do ledger preserva essa direção, deixando nomes e caminhos para a fase de implementação.

## Alternativas Consideradas

| Alternativa | Motivo da rejeição | Trade-off |
|---|---|---|
| Trigger do banco para acordar o processamento | O MySQL não notifica diretamente o processo externo; improvisar arquivo ou chamada de endpoint seria inadequado. | Poderia parecer mais reativo, mas misturaria lógica de integração externa com trigger SQL e não resolveria o consumo HTTP. |
| Worker dentro do mesmo processo da API | Uma reinicialização da API também interromperia o worker, contrariando a necessidade de isolamento operacional. | Teria implantação mais simples, mas perderia independência de ciclo de vida e de conexão por processo. |

## Consequências

### Positivas

- O intervalo de 2 segundos atende folgadamente à meta de menos de 10 segundos no cenário normal.
- O worker pode reiniciar e escalar de forma independente da API.
- O desenho reutiliza o MySQL e o Prisma existentes, sem listener ou infraestrutura de fila adicional.
- A ordenação por `created_at` é determinística enquanto houver um único worker.

### Negativas

- O polling introduz uma latência mínima e trabalho periódico mesmo quando não há eventos.
- A capacidade inicial fica limitada ao processamento do single-worker.
- Não há garantia de ordenação global se o sistema for paralelizado no futuro.
- Escala, particionamento e estratégia de locks exigirão decisão posterior.

## Relações e referências

- A Outbox consumida por este processo está no [ADR-001](ADR-001-outbox-no-mysql.md).
- Retry, timeout, DLQ e replay estão no [ADR-003](ADR-003-retry-com-backoff-e-dlq.md).
- O reuso da configuração Prisma e dos módulos existentes está no [ADR-006](ADR-006-reuso-dos-padroes-do-projeto.md).
- A direção de alto nível será resumida no [RFC](../RFC.md) e os fluxos detalhados no [FDD](../FDD.md).
- Evidências primárias: [TRANSCRICAO.md](../../TRANSCRICAO.md), `[09:09–09:13]` e `[09:29–09:30]`.
