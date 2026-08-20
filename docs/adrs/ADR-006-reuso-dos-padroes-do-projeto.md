# ADR-006: Reuso dos padrões do projeto

## Status

Aceito para o design. A implementação da funcionalidade não faz parte desta entrega.

## Contexto

O código-base organiza cada domínio em módulos com controller, service, repository, routes e schemas. Também possui padrões compartilhados para Prisma, autenticação JWT, papéis `ADMIN` e `OPERATOR`, `AppError`, códigos de erro, validação Zod, middleware de erro e logger Pino. Essas estruturas foram apresentadas por Bruno e Larissa entre 09:27 e 09:30 e confirmadas no ledger por inspeção dos caminhos reais do clone.

A funcionalidade deve integrar a mudança de status no `OrderService.changeStatus` recebendo o transaction client atual, sem criar uma arquitetura paralela. O worker separado usará o mesmo banco e a mesma stack, com uma instância de Prisma própria do processo. O replay da DLQ reutilizará `requireRole('ADMIN')`, e os erros do novo módulo seguirão o prefixo `WEBHOOK_`, conforme 09:28–09:30 e 09:35–09:36.

## Decisão

Implementar o design futuro dentro dos padrões existentes:

- organizar o domínio sob a pasta sugerida `src/modules/webhooks`, com controller, service, repository, routes e schemas;
- registrar as rotas no router principal e manter a autenticação existente para o CRUD;
- usar Zod nos contratos e `AppError` com códigos `WEBHOOK_*`, aproveitando o middleware de erro existente;
- usar Pino e suas regras de redaction, sem introduzir outro logger;
- integrar `OrderService.changeStatus` por uma função que receba o transaction client, conforme a proposta R-002 do ledger;
- executar o worker como processo separado com um `PrismaClient` próprio apontando para o mesmo banco;
- proteger o replay com o papel `ADMIN` existente e registrar o ator.

Os nomes de arquivos, os modelos novos e os contratos HTTP são desenho para o FDD, não código já presente no repositório.

## Alternativas Consideradas

| Alternativa | Motivo da rejeição | Trade-off |
|---|---|---|
| Criar um serviço ou stack independente para webhooks | A reunião decidiu reusar ao máximo a stack, o banco e os padrões já existentes; o time não queria infraestrutura adicional para o cenário atual. | Isolaria a evolução do webhook, mas criaria novas fronteiras operacionais, autenticação e observabilidade. |
| Colocar a lógica diretamente no controller de pedidos, sem módulo | Não seguiria a organização de domínios da codebase e misturaria configuração, envio e mudança de status. | Poderia reduzir arquivos iniciais, mas aumentaria acoplamento e dificultaria testes e evolução. |
| Criar códigos e tratamento de erro ad hoc | O projeto já possui `AppError`, middleware centralizado e códigos estáveis. | Seria rápido localmente, mas produziria respostas e logs inconsistentes com os demais módulos. |

## Consequências

### Positivas

- A nova documentação fica alinhada ao fluxo real do projeto e reduz decisões de integração posteriores.
- Autenticação, autorização, validação, erros e logging permanecem consistentes com a API existente.
- O ponto transacional de `OrderService.changeStatus` fica explícito sem alterar o código nesta entrega.
- A equipe poderá implementar a funcionalidade usando a mesma organização de módulos e o mesmo Prisma/MySQL.

### Negativas

- O design fica acoplado às fronteiras e convenções atuais do projeto.
- Novos modelos, rotas, repositórios, worker e testes ainda precisarão ser criados na implementação futura.
- A redaction atual do logger deverá ser revisada para garantir que segredos de webhook nunca sejam registrados.
- A separação do worker exige operação de um segundo processo e ciclo de conexão Prisma próprio.

## Relações e referências

- A atomicidade da integração está no [ADR-001](ADR-001-outbox-no-mysql.md).
- Processo separado e polling estão no [ADR-002](ADR-002-worker-separado-e-polling-de-2s.md).
- Retry e DLQ estão no [ADR-003](ADR-003-retry-com-backoff-e-dlq.md).
- Segurança e semântica de entrega estão nos [ADR-004](ADR-004-hmac-sha256-e-segredo-por-endpoint.md) e [ADR-005](ADR-005-at-least-once-e-x-event-id.md).
- A visão de alto nível ficará no [RFC](../RFC.md) e a integração detalhada no [FDD](../FDD.md).
- Evidências primárias: [TRANSCRICAO.md](../../TRANSCRICAO.md), `[09:27–09:30]`, `[09:35–09:36]` e `[09:40–09:41]`; caminhos de código registrados no ledger da Fase 1.
