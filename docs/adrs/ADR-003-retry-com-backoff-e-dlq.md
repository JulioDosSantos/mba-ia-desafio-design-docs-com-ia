# ADR-003: Retry com backoff e dead-letter queue

## Status

Aceito para o design. A implementação da funcionalidade não faz parte desta entrega.

## Contexto

Um consumidor pode estar indisponível ou responder depois do limite aceito. A reunião definiu que timeout HTTP de 10 segundos é falha e deve acionar retry. Diego e Larissa escolheram cinco tentativas com intervalos de aproximadamente 1 minuto, 5 minutos, 30 minutos, 2 horas e 12 horas, em uma janela total próxima de 15 horas, entre 09:15 e 09:17.

Uma falha que persiste após a janela precisa sair do fluxo normal sem permanecer indefinidamente na Outbox. A equipe escolheu uma tabela separada `webhook_dead_letter`, com payload, motivo da falha e timestamp, e replay manual por `POST /admin/webhooks/dead-letter/:id/replay`, conforme 09:17–09:18. O replay exige `ADMIN` e auditoria do ator, conforme 09:35–09:36.

## Decisão

Aplicar timeout de 10 segundos e a sequência fechada de cinco tentativas com backoff de 1 minuto, 5 minutos, 30 minutos, 2 horas e 12 horas. Depois da última tentativa sem sucesso, persistir a falha na tabela separada `webhook_dead_letter`, preservando payload, motivo e timestamp para diagnóstico e reprocessamento.

Disponibilizar o replay administrativo indicado na reunião. O replay deve recolocar o evento na Outbox como pendente, exigir o papel `ADMIN` e registrar o ator. O FDD deverá detalhar os estados, a transição e os status HTTP sem apresentar esses detalhes ainda não definidos como fatos da reunião.

## Alternativas Consideradas

| Alternativa | Motivo da rejeição | Trade-off |
|---|---|---|
| Retry indefinido | Um cliente desaparecido poderia manter o evento pendurado para sempre. | Aumentaria a chance de recuperação automática, mas impediria uma conclusão operacional do evento. |
| Somente três tentativas | Seria insuficiente para uma indisponibilidade planejada de aproximadamente duas horas. | Reduziria carga e tempo de espera, mas sacrificaria a janela considerada aceitável pela equipe. |
| Marcar apenas `failed` na própria Outbox | A equipe preferiu uma tabela separada como evidência para debug e reprocessamento, mantendo a leitura da Outbox principal mais limpa. | Evitaria uma tabela adicional, mas misturaria falhas finais ao fluxo normal e dificultaria o tratamento específico de DLQ. |

## Consequências

### Positivas

- O sistema tolera indisponibilidades temporárias sem bloquear a transação original do pedido.
- A janela de aproximadamente 15 horas cobre o cenário de manutenção planejada citado na reunião.
- A DLQ torna a falha final observável e permite replay controlado.
- Timeout, número de tentativas e backoff ficam explícitos para operação e consumidores.

### Negativas

- A fila pode acumular backlog durante indisponibilidade ou lentidão do consumidor.
- Cada tentativa adicional aumenta tráfego outbound e o tempo até a falha final.
- A DLQ exige retenção, acesso administrativo e auditoria, além do armazenamento da Outbox.
- Rate limiting outbound não foi decidido e permanece uma pendência separada.

## Relações e referências

- O worker que aplica esse fluxo está no [ADR-002](ADR-002-worker-separado-e-polling-de-2s.md).
- A persistência atômica inicial está no [ADR-001](ADR-001-outbox-no-mysql.md).
- O papel `ADMIN` e o reuso de autenticação estão no [ADR-006](ADR-006-reuso-dos-padroes-do-projeto.md).
- O resumo de alto nível ficará no [RFC](../RFC.md) e o fluxo detalhado no [FDD](../FDD.md).
- Evidências primárias: [TRANSCRICAO.md](../../TRANSCRICAO.md), `[09:15–09:18]`, `[09:35–09:36]` e `[09:42]`.
