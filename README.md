# Processo de produção — Design Docs Gerados por IA

## Sobre o desafio

Este desafio transforma uma reunião técnica em um pacote de design docs para um sistema de webhooks outbound que notifica mudanças de status de pedidos. A transcrição é a fonte primária das decisões; o código-base Node.js, TypeScript, Prisma e MySQL mostra onde a proposta deverá se integrar.

O resultado é documental: PRD, RFC, FDD, seis ADRs, Tracker e este registro de processo. A documentação deve ser acionável para uma implementação futura sem apresentar como existente aquilo que ainda não foi construído.

## Ferramentas de IA utilizadas

- **Codex, baseado em GPT-5:** leitura orientada da transcrição, código e materiais de apoio; organização do ledger; redação dos documentos; revisão de coerência, rastreabilidade e escopo.
- **PowerShell e `rg`:** inspeção determinística de arquivos, caminhos, headings, status do Git e contagens do Tracker. Essas ferramentas foram usadas para conferir o trabalho da IA, não para gerar decisões.

A IA foi usada como assistente de análise e redação. A revisão crítica comparou cada decisão com `TRANSCRICAO.md`, cada integração com caminhos existentes e cada proposta com o ledger. Decisões não fechadas continuam marcadas como proposta ou pendência; a aprovação final do pacote continua sendo uma etapa humana.

## Fontes e regras de trabalho

1. `TRANSCRICAO.md` foi tratada como fonte primária para decisões, requisitos, alternativas rejeitadas e limites.
2. O código existente foi consultado para confirmar fluxo de pedidos, autenticação, erros, logging, Prisma e entry-points.
3. O material do módulo de Design Docs orientou a separação entre produto, arquitetura, decisão e implementação.
4. A entrega ficou restrita a documentação. Não foram alterados `src/`, `prisma/`, `tests/`, dependências, configurações ou `TRANSCRICAO.md`.
5. Rotas, status HTTP, envelopes e nomes de componentes novos que não estavam fechados na reunião foram marcados como `PROPOSTA`; temas adiados ficaram como `PENDENCIA`.

## Workflow adotado

O fluxo foi executado em ciclos curtos, com revisão antes de avançar:

1. **Linha de base:** abrir o clone na branch `dev`, conferir `git status`, localizar `TRANSCRICAO.md`, inventariar o código e confirmar que nenhum arquivo de implementação seria alterado.
2. **Ledger de evidências:** separar fatos fechados, alternativas rejeitadas, pendências, propostas e evidências de código. Cada item recebeu uma fonte e uma localização verificável.
3. **ADRs primeiro:** registrar as seis decisões duráveis em MADR: Outbox MySQL, worker/polling, retry/DLQ, HMAC/segredo, at-least-once e reuso dos padrões existentes.
4. **RFC:** consolidar a direção arquitetural, trade-offs e questões abertas sem copiar o detalhamento de contratos e fluxos do FDD.
5. **FDD:** detalhar Outbox, worker, retry, DLQ, contratos HTTP, payload, headers, erros `WEBHOOK_*`, observabilidade e integração com arquivos reais do código.
6. **PRD:** consolidar problema, atores, métrica, escopo, requisitos funcionais e não funcionais, riscos e critérios de aceitação no nível de produto.
7. **Tracker:** varrer os documentos e mapear requisitos, decisões, riscos, contratos e caminhos de código para a transcrição ou o clone.
8. **README:** registrar o processo, os prompts e as correções feitas durante as revisões.
9. **Auditoria:** revisar links e seções, procurar campos incompletos e contradições, comparar valores críticos entre os documentos e conferir o diff contra o escopo documental.

Em cada ciclo, a IA produziu ou reorganizou um rascunho; a conferência seguinte usou fontes primárias, `rg`, `Test-Path`, `git status`, `git diff` e verificações específicas do documento. Um resultado de validação não substitui a leitura crítica: por exemplo, uma rota pode estar escrita corretamente e ainda precisar ser marcada como proposta.

## Prompts customizados

### 1. Extração de evidências

```text
Leia TRANSCRICAO.md e o código-base sem modificar nenhum arquivo. Monte uma tabela com ID, afirmação, documento de destino, tipo (TRANSCRICAO, CODE, PROPOSTA ou PENDENCIA), fonte exata, confiança e observação. Separe decisões fechadas, alternativas rejeitadas e questões abertas. Confirme os caminhos no código. Não invente nomes de tabelas, rotas, status HTTP ou componentes; quando a fonte não decidir algo, marque como PROPOSTA ou PENDENCIA. Ao final, liste contradições e perguntas que ainda exigem revisão.
```

### 2. ADR ancorado no ledger

```text
Com base somente no ledger de evidências, escreva um ADR em português para a decisão indicada. Use as seções Status, Contexto, Decisão, Alternativas Consideradas e Consequências, incluindo trade-offs positivos e negativos. Cite timestamps ou caminhos reais. Não transforme proposta em fato e não duplique o detalhamento do FDD. Se a decisão não estiver fechada, registre a pendência em vez de aceitá-la.
Decisão a documentar: uma única decisão do ledger
```

### 3. Auditoria do pacote

```text
Audite PRD, RFC, FDD, ADRs e Tracker sem editar arquivos. Verifique requisitos, métricas, exclusões, alternativas, questões abertas, links, contratos, matriz WEBHOOK_*, observabilidade, caminhos reais de código, cobertura do Tracker, timestamps, campos incompletos e escopo do diff. Compare entre os documentos polling, tentativas, backoff, timeout, payload, segredo, at-least-once e endpoints. Retorne PASS/FAIL com evidência e correções priorizadas. Não invente solução para uma pendência.
```

## Iterações e ajustes

Foram registrados quatro ajustes principais durante a produção e a revisão:

1. **`customer_id` e contrato REST:** a conversa não decidiu se o identificador ficaria no body ou no path. O RFC passou a manter isso como questão aberta, e o FDD escolheu uma forma coerente apenas como contrato `PROPOSTA`, sem atribuí-la à reunião.
2. **Fronteira entre RFC e FDD:** o RFC foi mantido em nível de direção e trade-offs; exemplos de payload, endpoints, estados, códigos `WEBHOOK_*` e integração com arquivos foram concentrados no FDD.
3. **Auditoria do FDD:** a matriz de erros `WEBHOOK_*`, a distinção entre erros já existentes e códigos novos, e a ausência de uma plataforma de tracing foram explicitadas para evitar que o design parecesse uma implementação existente.
4. **Auditoria do Tracker:** o stub foi substituído por 126 linhas rastreáveis. Na revisão, o RF-04 foi corrigido para não apresentar a retenção de deliveries como fato da transcrição; referências de seções foram ajustadas para os títulos reais. Uma segunda auditoria corrigiu a taxonomia da coluna `Fonte`: propostas e pendências permaneceram em `Tipo`, enquanto a origem passou a usar somente `TRANSCRICAO` ou `CODIGO`. O resultado final tem 104 linhas `TRANSCRICAO` (82,5%), todas com citações pontuais no formato `[hh:mm] Nome`, e 22 linhas `CODIGO` com caminhos verificáveis.

## Como navegar a entrega

A ordem sugerida acompanha a altura de cada documento:

1. [Transcrição da reunião](TRANSCRICAO.md) — fonte primária das decisões.
2. [PRD](docs/PRD.md) — problema, público, escopo, métricas e requisitos.
3. [RFC](docs/RFC.md) — proposta arquitetural, alternativas e questões abertas.
4. ADRs: [ADR-001](docs/adrs/ADR-001-outbox-no-mysql.md), [ADR-002](docs/adrs/ADR-002-worker-separado-e-polling-de-2s.md), [ADR-003](docs/adrs/ADR-003-retry-com-backoff-e-dlq.md), [ADR-004](docs/adrs/ADR-004-hmac-sha256-e-segredo-por-endpoint.md), [ADR-005](docs/adrs/ADR-005-at-least-once-e-x-event-id.md) e [ADR-006](docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md) — decisões isoladas.
5. [FDD](docs/FDD.md) — fluxos, contratos, erros, resiliência e integração com o código.
6. [Tracker](docs/TRACKER.md) — rastreabilidade transversal de requisitos, decisões, riscos, contratos e caminhos.

O RFC explica por que a direção foi escolhida; os ADRs registram decisões duráveis; o FDD descreve como implementar depois. Quando houver conflito aparente, consulte primeiro o Tracker e a fonte indicada na coluna `Localização`.

## Limites conhecidos

Rate limiting outbound, escala para múltiplos workers, ordenação global, arquivamento, email de falhas, dashboard visual e webhook inbound permanecem fora desta entrega ou postergados. O repositório contém a especificação e as evidências necessárias para uma próxima implementação, mas não contém ainda o módulo de webhooks, o worker, os modelos Prisma ou os testes específicos dessa feature.
