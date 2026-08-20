# RFC — Webhooks outbound para mudanças de status de pedidos

## Metadados

| Campo | Valor |
|---|---|
| Autor | Julio dos Santos |
| Status | Proposto para revisão |
| Data | 2026-08-19 |
| Revisores | Larissa, Tech Lead; Marcos, Product Manager; Bruno, Engenheiro Pleno do time de Pedidos; Diego, Engenheiro Sênior do time de Plataforma; Sofia, Engenheira de Segurança |
| Escopo | Notificação outbound de mudanças de status de pedidos para clientes B2B |

## TL;DR

Propomos substituir o polling periódico dos clientes por webhooks outbound de mudanças de status. A mudança do pedido continuará sendo confirmada pela transação atual e, na mesma transação, um evento será registrado em uma Outbox no MySQL. Um worker em processo separado consultará os eventos pendentes por polling a cada 2 segundos e fará as entregas HTTP.

A proposta mantém o sistema desacoplado da disponibilidade dos consumidores: falhas e timeout de 10 segundos serão tratados por cinco tentativas com backoff de 1 minuto, 5 minutos, 30 minutos, 2 horas e 12 horas; falhas finais seguirão para uma DLQ com replay administrativo. A entrega será at-least-once, com UUID em `X-Event-Id` para que o consumidor deduplicate. A comunicação exigirá HTTPS e HMAC-SHA256 com segredo exclusivo por endpoint e rotação do segredo anterior por 24 horas.

Essa direção atende à meta operacional de menos de 10 segundos no cenário normal sem adicionar Redis ou tornar a chamada externa parte da transação crítica. O RFC registra a direção para revisão; o detalhamento implementável de contratos, persistência, erros e fluxos pertence ao [FDD](FDD.md).

## Contexto e problema

Atlas Comercial, MaxDistribuição e Nova Cargo querem saber quando os pedidos mudam de status. Hoje, os clientes consultam `GET /orders` repetidamente para descobrir a mudança. Esse polling deixa a integração lenta e cara para os clientes, e a Atlas sinalizou risco de migração se a capacidade não for entregue até o fim do trimestre. Para esses clientes, qualquer entrega abaixo de 10 segundos já é considerada tempo real.

O problema não é apenas transportar uma mensagem. A mudança de status já ocorre em uma transação que atualiza o pedido, registra o histórico e pode ajustar estoque. Fazer uma chamada HTTP síncrona nesse caminho permitiria que um consumidor lento ou indisponível travasse outros pedidos ou provocasse rollback. Ao mesmo tempo, um evento não pode ser perdido depois que a transação do pedido for confirmada.

O escopo discutido é exclusivamente outbound: a plataforma envia notificações, e os clientes não enviam webhooks para a plataforma nesta entrega. O CRUD de configuração, o histórico limitado de entregas e o replay administrativo são capacidades de apoio à integração, não um portal visual completo.

## Proposta técnica

### Direção arquitetural

A plataforma registrará o evento em uma Outbox no MySQL dentro da mesma transação da mudança de status. Assim, o commit do pedido e a existência do evento formam uma unidade; em caso de rollback, o evento também desaparece. O filtro de status será aplicado na inserção. Se nenhum endpoint do cliente estiver interessado naquele status, não será criado evento desnecessário.

Um worker separado da API consumirá a Outbox por polling de 2 segundos, usando o mesmo banco e a mesma stack Prisma, mas com ciclo de vida próprio. O cenário inicial será single-worker, processando os eventos pendentes mais antigos. Isso oferece ordenação implícita por pedido nesse cenário, sem prometer ordenação global para um futuro com múltiplos workers.

O worker entregará o snapshot do evento ao endpoint HTTPS configurado. O corpo será autenticado por HMAC-SHA256 com segredo próprio de cada endpoint. `X-Event-Id`, `X-Signature`, `X-Timestamp` e `X-Webhook-Id` formarão parte do envelope de entrega. A rotação manterá o segredo anterior válido por 24 horas para permitir a migração do cliente.

Uma falha de transporte, uma resposta não bem-sucedida ou o timeout de 10 segundos levará o evento ao fluxo de retry. Serão usadas cinco tentativas com backoff de 1 minuto, 5 minutos, 30 minutos, 2 horas e 12 horas. Depois da janela, o evento irá para uma DLQ separada, preservando evidências para diagnóstico e permitindo replay manual por endpoint administrativo com papel `ADMIN` e registro do ator.

O contrato adotará at-least-once. O mesmo UUID será criado quando o evento entrar na Outbox e seguirá no payload e em `X-Event-Id`; o consumidor será responsável por deduplicar. Essa decisão torna explícita a possibilidade de duplicatas e evita uma promessa de exactly-once que exigiria coordenação entre a plataforma e cada cliente.

### Fronteira com o FDD

Este RFC decide a direção e os motivos. O FDD deverá transformar a direção em desenho implementável, incluindo modelos e estados da Outbox/DLQ, contratos HTTP propostos, exemplos de payload, status e códigos `WEBHOOK_*`, seleção em lote, métricas, logs, tracing e os caminhos de integração no código. O RFC não fixa a localização de `customer_id` no body ou no path, nem inventa status HTTP ou uma plataforma de tracing que não foram decididos na reunião.

## Alternativas consideradas

| Alternativa | Por que foi descartada | Trade-off principal |
|---|---|---|
| HTTP síncrono dentro da transação de mudança de status | Um endpoint lento ou fora do ar tornaria a transação do pedido dependente da disponibilidade externa. | Seria o caminho mais direto, mas sacrificaria isolamento, disponibilidade e previsibilidade do fluxo de pedidos. |
| Redis Streams ou outra fila externa | Exigiria infraestrutura adicional e foi considerada complexidade desproporcional para o time e o cenário atual. | Poderia oferecer uma infraestrutura dedicada de consumo, mas aumentaria operação, dependências e custo antes de existir necessidade comprovada. |
| Trigger do MySQL para acordar o envio | Trigger executa SQL e não notifica adequadamente um processo HTTP externo; improvisar arquivo ou endpoint seria frágil. | Poderia reduzir o intervalo de polling em teoria, mas não resolveria o consumo outbound e misturaria integração externa com lógica SQL. |

A Outbox no MySQL foi escolhida porque preserva atomicidade sem introduzir um novo componente de infraestrutura. O worker separado mantém a entrega fora da transação e oferece um ponto claro para retry, DLQ e observabilidade.

## Questões em aberto

1. **Rate limiting outbound:** a reunião decidiu observar o volume e decidir depois. O RFC não fixa quota, janela, algoritmo ou política por cliente. O backlog, o número de falhas e o volume de mudanças deverão informar essa decisão.

2. **Escala e ordenação:** o single-worker e a ordenação por `created_at` são suficientes para o cenário atual, mas não há garantia de ordenação global se a plataforma for paralelizada. Particionamento por `order_id` e locks pessimistas foram deixados para uma evolução futura.

3. **Contrato de configuração:** a reunião confirmou CRUD autenticado e a necessidade de um identificador de cliente, mas não escolheu se `customer_id` ficará no body ou no path. A convenção REST, os status HTTP, a paginação de deliveries e o envelope de erro serão propostas no FDD e devem permanecer identificadas como desenho, não como decisão histórica.

Essas questões não bloqueiam a direção arquitetural, mas precisam ser resolvidas antes da implementação. Inbound webhook, email de falha, dashboard visual e arquivamento automático após aproximadamente 30 dias permanecem fora do escopo desta entrega.

## Impacto e riscos

### Impactos esperados

- **Clientes:** deixam de depender de polling contínuo e recebem notificações assinadas, com uma meta operacional de menos de 10 segundos no cenário normal.
- **Pedidos:** a mudança de status não espera por HTTP externo e não perde o evento quando a transação confirma.
- **Operação:** surge um processo worker para operar, uma Outbox para acompanhar, uma política finita de retry e uma DLQ para investigar e reprocessar.
- **Segurança:** cada endpoint passa a ter segredo próprio, assinatura HMAC, HTTPS obrigatório e uma janela controlada de rotação.
- **Entrega:** a estimativa discutida é de três sprints, incluindo dois dias úteis reservados para a revisão de segurança de Sofia.

### Riscos e mitigação arquitetural

| Risco | Impacto | Mitigação prevista |
|---|---|---|
| Consumidor indisponível ou lento | Backlog, latência acima da meta e falhas finais | Timeout de 10 segundos, cinco tentativas com backoff, DLQ e replay administrativo; a evolução de rate limiting permanece em aberto. |
| Duplicatas no consumidor | Processamento repetido de uma mudança | Semântica at-least-once documentada e deduplicação por `X-Event-Id`. |
| Vazamento ou rotação inadequada de segredo | Falsificação ou exposição de eventos | Segredo por endpoint, HMAC-SHA256, HTTPS, validade paralela de 24 horas e revisão de segurança. |
| Crescimento do backlog ou necessidade de paralelismo | Limitação de throughput e perda de ordenação global | Começar com single-worker explícito, observar idade/backlog e decidir particionamento ou locks com base em evidência. |
| Falta de limite de envio | Sobrecarga de um cliente durante picos | Registrar rate limiting como questão aberta e observar o comportamento antes de escolher a política. |

## Revisão e decisões relacionadas

Este RFC deve ser revisado pelos cinco participantes da reunião nos papéis registrados nos metadados. A revisão de segurança de Sofia precisa ocorrer antes do deploy, com a reserva de dois dias úteis discutida na reunião. As decisões pontuais que sustentam esta proposta são:

- [ADR-001 — Outbox no MySQL na transação do pedido](adrs/ADR-001-outbox-no-mysql.md)
- [ADR-002 — Worker separado e polling de 2 segundos](adrs/ADR-002-worker-separado-e-polling-de-2s.md)
- [ADR-003 — Retry com backoff e dead-letter queue](adrs/ADR-003-retry-com-backoff-e-dlq.md)
- [ADR-004 — HMAC-SHA256 e segredo por endpoint](adrs/ADR-004-hmac-sha256-e-segredo-por-endpoint.md)
- [ADR-005 — At-least-once e `X-Event-Id`](adrs/ADR-005-at-least-once-e-x-event-id.md)
- [ADR-006 — Reuso dos padrões do projeto](adrs/ADR-006-reuso-dos-padroes-do-projeto.md)

## Referências

- [TRANSCRICAO.md](../TRANSCRICAO.md), especialmente `[09:00–09:10]` para problema e direção inicial, `[09:15–09:26]` para retry, DLQ e semântica de entrega, e `[09:31–09:47]` para configuração, segurança e prazo.
- [ADR-001](adrs/ADR-001-outbox-no-mysql.md) a [ADR-006](adrs/ADR-006-reuso-dos-padroes-do-projeto.md).
