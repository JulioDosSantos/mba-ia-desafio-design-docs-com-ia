# PRD — Sistema de Webhooks de Notificação de Pedidos

Versão: 1.0
Data: 2026-08-19
Responsável: Julio dos Santos
Status: Proposto para revisão

---

## Resumo

O Sistema de Webhooks de Notificação de Pedidos permitirá que clientes B2B recebam mudanças de status dos seus pedidos sem consultar repetidamente a plataforma. A primeira entrega atende Atlas Comercial, MaxDistribuição e Nova Cargo, que hoje dependem de polling e consideram qualquer notificação abaixo de 10 segundos como tempo real.

A feature enviará notificações outbound para endpoints configurados pelos clientes. O sistema manterá a atualização do pedido independente da disponibilidade do consumidor, oferecerá segurança por HTTPS e HMAC-SHA256, tentará novamente entregas temporariamente indisponíveis e disponibilizará histórico e recuperação administrativa de falhas finais.

## Contexto e problema

### Público-alvo e atores

- **Clientes B2B:** Atlas Comercial, MaxDistribuição e Nova Cargo, que precisam reagir rapidamente a alterações nos pedidos.
- **Consumidores de webhook:** sistemas técnicos mantidos pelos clientes para receber, validar e deduplicar notificações.
- **Operadores e administradores:** pessoas autenticadas que configuram endpoints, consultam entregas e, quando autorizadas, fazem replay de falhas.
- **Equipe do sistema:** produto, pedidos, plataforma e segurança, responsáveis por operar a entrega e revisar a evolução.

### Cenários de uso principais

1. Um cliente cadastra um endpoint, escolhe os status de pedido que deseja acompanhar e recebe o segredo gerado para validar as notificações.
2. Um pedido muda de status e o sistema registra uma notificação para cada endpoint interessado, sem tornar o consumidor externo parte da atualização do pedido.
3. O consumidor recebe a notificação, valida HTTPS e assinatura, e deduplica o evento quando uma mesma entrega aparece mais de uma vez.
4. Um operador consulta as últimas entregas para entender sucesso, falha, payload, resposta e tempo de resposta.
5. Um administrador reprocessa uma falha final depois que o cliente corrige sua indisponibilidade.

### Problemas priorizados

| Problema | Impacto | Prioridade |
|---|---|---|
| Clientes precisam consultar pedidos repetidamente para descobrir mudanças. | Integração lenta, custo desnecessário e experiência operacional ruim. | Alta |
| A Atlas pode migrar para um concorrente se não houver notificação até o fim do trimestre. | Risco comercial e perda de cliente. | Alta |
| Uma chamada externa síncrona poderia bloquear a atualização de outros pedidos. | Indisponibilidade ou degradação do fluxo principal de pedidos. | Alta |
| Falhas de consumidores precisam ser recuperáveis e auditáveis. | Eventos perdidos, suporte manual e dificuldade de diagnóstico. | Média |

### Onde a feature será implantada

A feature será adicionada ao Order Management System existente e atenderá somente notificações outbound de mudanças de status. Não será criado um sistema inbound de webhooks nem um portal visual nesta entrega.

---

## Objetivos e métricas

| Objetivo | Métrica | Meta |
|---|---|---|
| Substituir a descoberta baseada em polling por notificação de status. | Latência entre a confirmação da mudança e a entrega ao endpoint no cenário normal. | Menos de 10 segundos; o indicador deve ser confirmado após a implementação. |
| Evitar divergência entre mudança confirmada e notificação elegível. | Eventos elegíveis sem registro persistido após uma mudança confirmada. | Zero perdas causadas pela confirmação da mudança; a medição será validada nos testes. |
| Permitir recuperação de indisponibilidade temporária do consumidor. | Entregas que seguem a política de retry ou chegam à fila de falhas recuperáveis. | Toda falha após a janela de tentativas deve ficar disponível para investigação e replay administrativo. |
| Proteger a comunicação com os consumidores. | Entregas outbound que usam HTTPS e assinatura com segredo específico do endpoint. | 100% das entregas; configurações HTTP devem ser recusadas. |

---

## Escopo

### Incluso

- Cadastro, consulta, edição, desativação e remoção de endpoints por cliente.
- Geração do segredo na criação e rotação com o segredo anterior válido por 24 horas.
- Seleção dos status/eventos que cada endpoint deseja receber.
- Registro assíncrono e atômico de notificações elegíveis.
- Entrega outbound com HTTPS, HMAC-SHA256, timeout e identificador de evento.
- Cinco tentativas com backoff de 1 minuto, 5 minutos, 30 minutos, 2 horas e 12 horas.
- Fila de falhas finais, replay administrativo e auditoria do ator do replay.
- Histórico das últimas 100 entregas com resultado, payload, resposta e tempo de resposta.

### Fora de escopo

- Recebimento de webhooks dos clientes pela plataforma.
- Email automático sobre falhas de endpoint.
- Dashboard visual, portal completo ou aplicação frontend para acompanhar entregas.
- Arquivamento automático de entregas depois de aproximadamente 30 dias.
- Rate limiting outbound, que será observado e decidido em uma evolução posterior.
- Escala para múltiplos workers, particionamento, locks e garantia de ordenação global.

---

## Requisitos funcionais

#### RF-01 — Cadastrar endpoint de webhook

O sistema deve permitir que um cliente autenticado cadastre um endpoint e receba um segredo gerado pela plataforma na criação.

**Fluxo principal**

- Usuário autenticado informa o cliente, a URL HTTPS e os status/eventos desejados.
- O sistema valida a configuração, gera um segredo exclusivo e retorna os dados necessários para a primeira integração.

**Fluxos alternativos e exceções**

- O cliente pode cadastrar mais de um endpoint, desde que cada cadastro tenha configuração própria.
- O segredo não deve ser exposto novamente em listagens comuns.

**Erros previstos**

- Configuração inválida ou URL que não seja HTTPS.
- Usuário não autenticado ou cliente inexistente.

**Prioridade:** alta

---

#### RF-02 — Listar endpoints do cliente

O sistema deve permitir consultar os endpoints configurados para um cliente, sem revelar segredos.

**Fluxo principal**

- Usuário autenticado solicita a configuração de um cliente.
- O sistema retorna endpoints, status ativo, URL e eventos selecionados.

**Fluxos alternativos e exceções**

- Cliente sem endpoints recebe uma lista vazia.
- A autorização específica entre usuário e cliente precisa ser confirmada antes da implementação, pois a reunião confirmou autenticação normal, mas não definiu esse vínculo.

**Erros previstos**

- Identificação de cliente inválida.
- Usuário sem autenticação.

**Prioridade:** alta

---

#### RF-03 — Editar configuração do endpoint

O sistema deve permitir atualizar URL, status/eventos de interesse e ativação do endpoint sem exigir uma nova configuração completa.

**Fluxo principal**

- Usuário autenticado seleciona um endpoint existente.
- O sistema valida os novos dados e salva a configuração atualizada.

**Fluxos alternativos e exceções**

- Alterar somente os eventos não deve gerar uma nova entrega retroativa.
- A alteração do segredo deve ocorrer pela operação de rotação, preservando a janela de transição.

**Erros previstos**

- Endpoint inexistente.
- URL HTTP, evento inválido ou configuração inconsistente.

**Prioridade:** alta

---

#### RF-04 — Remover ou desativar endpoint

O sistema deve permitir remover ou desativar um endpoint para impedir novas entregas a ele.

**Fluxo principal**

- Usuário autenticado solicita a remoção ou desativação.
- O sistema impede novas notificações para a configuração inativa.

**Fluxos alternativos e exceções**

- O histórico já registrado deve permanecer consultável conforme a política de retenção.
- A escolha entre remoção lógica e remoção definitiva é uma decisão de produto/modelo a ser confirmada na implementação.

**Erros previstos**

- Endpoint inexistente.
- Usuário sem autenticação.

**Prioridade:** média

---

#### RF-05 — Validar configuração segura

O sistema deve aceitar somente configurações válidas, incluindo URL HTTPS, lista de eventos suportada e dados necessários para assinatura.

**Fluxo principal**

- A configuração é validada antes de ser salva.
- Configurações inválidas são recusadas sem produzir novas notificações.

**Fluxos alternativos e exceções**

- Uma configuração HTTP não pode ser convertida automaticamente para HTTPS.
- Um endpoint sem segredo válido não pode receber entrega assinada.

**Erros previstos**

- URL inválida ou sem HTTPS.
- Eventos/status inválidos.
- Segredo ausente ou erro de configuração de assinatura.

**Prioridade:** alta

---

#### RF-06 — Selecionar status/eventos por endpoint

O sistema deve permitir que cada endpoint escolha quais mudanças de status deseja receber.

**Fluxo principal**

- O cliente informa a lista de status/eventos durante o cadastro ou edição.
- Somente mudanças que correspondem ao filtro do endpoint tornam-se elegíveis para entrega.

**Fluxos alternativos e exceções**

- Endpoints do mesmo cliente podem ter filtros diferentes.
- Alterar o filtro afeta eventos futuros, não eventos já registrados.

**Erros previstos**

- Status não suportado ou lista inválida.
- Configuração sem nenhum evento permitido.

**Prioridade:** alta

---

#### RF-07 — Registrar notificação de forma atômica

Quando uma mudança de status for confirmada, o sistema deve registrar a notificação elegível na mesma unidade transacional da mudança.

**Fluxo principal**

- O pedido muda de status com sucesso.
- O evento correspondente é registrado com UUID e snapshot do estado da mudança.

**Fluxos alternativos e exceções**

- Se a mudança sofrer rollback, não deve existir notificação correspondente.
- Se o registro da notificação falhar, a mudança do pedido também não pode ser confirmada.

**Erros previstos**

- Falha de persistência que provoque rollback da operação.
- Nenhum endpoint interessado, caso em que não há notificação para registrar.

**Prioridade:** alta

---

#### RF-08 — Evitar eventos sem consumidor interessado

O sistema não deve criar trabalho de entrega quando nenhum endpoint ativo do cliente estiver inscrito no status/evento ocorrido.

**Fluxo principal**

- O sistema avalia os filtros ativos no momento da mudança.
- Se não houver correspondência, conclui a mudança sem criar notificação outbound.

**Fluxos alternativos e exceções**

- Se ao menos um endpoint tiver interesse, a notificação deve ser criada para os destinos elegíveis.
- A existência de outros endpoints sem interesse não deve bloquear os interessados.

**Erros previstos**

- Falha ao consultar a configuração durante a transação, que deve impedir uma confirmação inconsistente.

**Prioridade:** alta

---

#### RF-09 — Processar entregas em processo separado

O sistema deve processar notificações pendentes de forma assíncrona, em processo separado da API, usando polling de 2 segundos.

**Fluxo principal**

- O processo de entrega identifica notificações pendentes.
- Ele envia cada notificação ao endpoint elegível e registra o resultado.

**Fluxos alternativos e exceções**

- Reiniciar a API não deve depender de uma chamada síncrona a consumidores externos.
- O cenário inicial usa um único worker e não promete ordenação global.

**Erros previstos**

- Worker indisponível, endpoint fora do ar ou backlog crescente.

**Prioridade:** alta

---

#### RF-10 — Aplicar timeout e política de retry

O sistema deve tratar timeout de 10 segundos, respostas não bem-sucedidas e falhas de rede com cinco tentativas e backoff definido.

**Fluxo principal**

- A primeira falha é registrada com sua causa.
- As tentativas seguintes ocorrem após 1 minuto, 5 minutos, 30 minutos, 2 horas e 12 horas, até a janela final.

**Fluxos alternativos e exceções**

- Uma resposta bem-sucedida encerra a entrega sem novas tentativas.
- O consumidor pode receber duplicata se a plataforma não souber se uma tentativa foi processada antes do timeout.

**Erros previstos**

- Timeout, resposta não-2xx ou falha de conexão.
- Falha final após esgotar as cinco tentativas.

**Prioridade:** alta

---

#### RF-11 — Persistir falhas finais e permitir replay administrativo

O sistema deve colocar falhas finais em uma fila de falhas recuperável e permitir que um administrador solicite replay com auditoria do ator.

**Fluxo principal**

- A falha após a última tentativa fica disponível com motivo, payload e timestamp.
- Um usuário com papel administrativo solicita o replay e o evento retorna ao fluxo de entrega.

**Fluxos alternativos e exceções**

- Usuário sem papel administrativo não pode fazer replay.
- Um item inexistente ou já reprocessado não deve gerar uma segunda operação ambígua.

**Erros previstos**

- Falta de permissão, item inexistente ou configuração de destino inválida.
- Falha ao recolocar o evento para processamento.

**Prioridade:** alta

---

#### RF-12 — Consultar histórico das entregas

O sistema deve disponibilizar as últimas 100 entregas de um endpoint, incluindo resultado, payload, resposta e tempo de resposta, além do identificador do evento e metadados necessários para investigação.

**Fluxo principal**

- Usuário autenticado seleciona um endpoint.
- O sistema retorna no máximo as 100 entregas mais recentes e seus resultados.

**Fluxos alternativos e exceções**

- Endpoint sem entregas retorna uma coleção vazia.
- Segredos não devem aparecer no histórico; o conteúdo exposto deve respeitar a política de segurança.

**Erros previstos**

- Endpoint inexistente.
- Usuário sem autenticação.

**Prioridade:** média

---

## Requisitos não funcionais

### Performance

- A meta de produto é notificar mudanças em menos de 10 segundos no cenário normal; a confirmação deve ser feita com medição após a implementação.
- O ciclo de busca do processo de entrega deve ocorrer a cada 2 segundos.
- O endpoint externo não pode manter uma tentativa aberta por mais de 10 segundos.

### Disponibilidade e isolamento

- A indisponibilidade de um consumidor não pode bloquear a confirmação de pedidos de outros clientes.
- Falhas temporárias devem ser absorvidas pela política de retry; falhas finais devem permanecer recuperáveis na DLQ.
- O sistema não promete ordenação global fora do cenário inicial de um único worker.

### Segurança e autorização

- URLs HTTP devem ser recusadas; entregas usam HTTPS.
- O corpo deve ser assinado com HMAC-SHA256 e cada endpoint deve ter segredo próprio.
- A rotação mantém o segredo antigo válido por 24 horas.
- O replay exige papel administrativo e registra o ator.
- O segredo não deve ser exposto em listagens, logs ou mensagens de erro.

### Integridade e semântica de entrega

- A entrega é at-least-once; duplicatas são possíveis.
- O consumidor deve deduplicar usando o identificador único da notificação.
- O payload deve ser um snapshot do estado no momento da mudança e não deve incluir itens do pedido.
- O tamanho máximo do payload é 64 KB; acima disso, o envio deve falhar.

### Observabilidade

- Devem existir indicadores de latência, sucesso/falha, retries, DLQ, idade da notificação pendente e backlog.
- O histórico deve permitir investigar resposta e tempo de resposta das últimas 100 entregas.
- Logs estruturados não podem expor segredos. A solução de tracing, se houver, será definida posteriormente, pois a reunião não escolheu uma plataforma.

### Compatibilidade

- A feature deve funcionar no Order Management System existente e manter compatibilidade com os clientes B2B consumidores de HTTPS/JSON.
- A evolução não deve exigir que os clientes adotem webhook inbound.
- O desenho deve manter o comportamento existente de atualização de pedidos e não introduzir dependência obrigatória de Redis.

---

## Arquitetura e abordagem

### Abordagem

A abordagem é assíncrona e orientada a eventos: a mudança de status confirma o negócio e registra uma notificação; um processo separado realiza a entrega e controla falhas. O MySQL existente será usado como base de consistência da Outbox, e os endpoints dos clientes serão os destinos externos.

### Componentes conceituais

- **Configuração de webhooks:** cadastro por cliente, seleção de status/eventos e rotação de segredo.
- **Captura transacional de eventos:** registra somente notificações elegíveis quando a mudança do pedido é confirmada.
- **Processo de entrega:** busca pendências, envia HTTPS, assina o corpo e aplica retry.
- **Histórico e fila de falhas:** permite consulta das últimas 100 entregas e replay de falhas finais.
- **Consumidor do cliente:** valida assinatura, processa o evento e deduplica por identificador.

### Integrações

- Fluxo de mudança de status do pedido.
- Banco e transação já usados pelo sistema de pedidos.
- Usuários autenticados e papel administrativo para operações de configuração e replay.
- Endpoints HTTPS dos clientes B2B.

---

## Decisões e trade-offs

### Decisão: processamento assíncrono com Outbox

- **Justificativa:** evita que um consumidor externo lento ou indisponível bloqueie a atualização do pedido e mantém o evento junto da transação confirmada.
- **Trade-off:** introduz latência eventual, consumo de armazenamento no banco e necessidade de operar um processo de entrega.

### Decisão: Outbox no MySQL existente

- **Justificativa:** preserva atomicidade sem adicionar Redis Streams ou outra infraestrutura ao cenário atual.
- **Trade-off:** o banco passa a armazenar e servir também o fluxo de notificações, exigindo observação de backlog e retenção.

### Decisão: at-least-once com identificador de evento

- **Justificativa:** permite retry sem exigir coordenação exactly-once entre a plataforma e cada consumidor.
- **Trade-off:** clientes precisam deduplicar e podem observar o mesmo evento mais de uma vez.

### Decisão: segurança por endpoint

- **Justificativa:** HMAC-SHA256, HTTPS e segredo exclusivo reduzem o risco de adulteração e limitam o impacto de um vazamento.
- **Trade-off:** cada cliente precisa armazenar e rotacionar seu segredo, incluindo uma janela de convivência de 24 horas.

---

## Dependências

### Técnica: fluxo de status e persistência do OMS

O sistema de pedidos precisa continuar confirmando transições e oferecendo a transação que permitirá registrar a notificação de forma atômica.

### Técnica: capacidade de executar o processo de entrega

O ambiente deve conseguir iniciar e monitorar um processo separado para polling, com acesso ao mesmo banco do OMS.

### Externa: endpoints dos clientes

Atlas, MaxDistribuição e Nova Cargo precisam fornecer endpoints HTTPS acessíveis, validar HMAC e implementar deduplicação por identificador de evento.

### Organizacional: revisão de segurança

Sofia deve ter pelo menos dois dias úteis para revisar geração de segredo, HMAC e demais controles antes do deploy.

### Organizacional: planejamento de entrega

A estimativa discutida é de três sprints, com a revisão de segurança incluída no final e confirmação de prazo junto aos clientes.

---

## Riscos e mitigação

### Consumidor externo indisponível por período prolongado

- **Probabilidade:** alta
- **Impacto:** alto; aumenta latência, backlog e falhas finais para o cliente.
- **Mitigação:**
  - timeout de 10 segundos;
  - cinco tentativas com backoff;
  - DLQ, histórico e métricas de idade/backlog.
- **Plano de contingência:** disponibilizar replay administrativo depois que o cliente restabelecer o endpoint.

### Entrega duplicada ou fora de ordem

- **Probabilidade:** alta
- **Impacto:** médio/alto; o consumidor pode processar uma mudança repetidamente ou receber eventos em ordem diferente no futuro.
- **Mitigação:**
  - `X-Event-Id` estável;
  - documentação de deduplicação;
  - declaração explícita de at-least-once e ausência de ordenação global.
- **Plano de contingência:** o consumidor deve descartar duplicatas; uma evolução de particionamento/ordenação será avaliada se o cenário exigir.

### Crescimento do backlog sem política de rate limiting

- **Probabilidade:** média
- **Impacto:** alto; pode consumir capacidade e degradar a experiência de entrega.
- **Mitigação:**
  - medir backlog, idade, falhas e latência;
  - começar com escopo single-worker conhecido;
  - manter rate limiting como decisão futura, sem inventar uma quota.
- **Plano de contingência:** priorizar uma decisão operacional de limite ou escala antes de ampliar o volume de clientes.

### Vazamento ou rotação inadequada de segredo

- **Probabilidade:** média
- **Impacto:** alto; pode permitir falsificação ou exposição de eventos.
- **Mitigação:**
  - segredo exclusivo por endpoint;
  - HMAC-SHA256 e HTTPS;
  - segredo anterior válido por 24 horas;
  - revisão de segurança e proteção contra exposição em logs.
- **Plano de contingência:** rotacionar o segredo, desativar temporariamente o endpoint afetado e registrar a ocorrência para auditoria.

### Atraso na entrega até o fim do trimestre

- **Probabilidade:** média
- **Impacto:** alto; a Atlas pode migrar para um concorrente.
- **Mitigação:**
  - planejar três sprints;
  - manter a revisão de segurança de dois dias úteis explícita;
  - validar primeiro o fluxo essencial de outbound, retry e histórico.
- **Plano de contingência:** comunicar o estado aos clientes e priorizar a entrega mínima acordada, sem incluir dashboard, email ou inbound.

---

## Critérios de aceitação

Checklist objetivo que define quando a feature está pronta:

- Os três clientes-alvo conseguem cadastrar endpoints e selecionar os status que desejam receber.
- Um endpoint recebe uma notificação outbound quando ocorre um status inscrito.
- Uma mudança confirmada e um registro de notificação elegível não divergem; se a operação sofrer rollback, nenhuma notificação correspondente é entregue.
- A latência do cenário normal é medida e fica abaixo de 10 segundos.
- URL HTTP é recusada, toda entrega usa HTTPS e o consumidor consegue validar HMAC-SHA256 com segredo próprio.
- A rotação mantém o segredo anterior válido por 24 horas sem expor segredo em listagem ou log.
- Timeout, falha de rede e resposta não-2xx seguem cinco tentativas com os intervalos definidos.
- Falhas finais ficam consultáveis na DLQ e um administrador consegue solicitar replay com auditoria do ator.
- O consumidor pode deduplicar pelo identificador único do evento e a documentação informa at-least-once.
- O histórico mostra no máximo as últimas 100 entregas com resultado, payload, resposta e tempo de resposta.
- Métricas e logs permitem acompanhar latência, sucesso/falha, retries, DLQ, idade e backlog sem expor segredos.
- Inbound webhook, email, dashboard visual e arquivamento automático não fazem parte da entrega aceita.

---

## Testes e validação

### Tipos de teste obrigatórios

- Testes unitários das regras de configuração, validação HTTPS, seleção de status, geração/rotação de segredo e classificação de falhas.
- Testes de integração do fluxo de mudança confirmada, rollback, criação de notificação elegível e ausência de evento quando não há endpoint interessado.
- Testes do processo de entrega para polling, timeout, resposta não-2xx, backoff, quinta tentativa, DLQ e replay com e sem papel administrativo.
- Testes de contrato de segurança para HMAC, HTTPS, headers, segredo anterior durante 24 horas e ausência de segredo em logs/listagens.
- Testes de idempotência do consumidor usando o mesmo identificador em tentativas repetidas.
- Teste de desempenho para medir a meta de menos de 10 segundos e acompanhar backlog/idade no cenário normal.
- Validação de aceitação com os fluxos de configuração, recebimento e consulta de histórico antes do deploy.

### Estratégia de validação

- Validar primeiro os fluxos críticos em ambiente de teste com endpoints controlados: sucesso, indisponibilidade, recuperação e replay.
- Medir a latência real após a implementação; a meta de menos de 10 segundos é um objetivo, não uma evidência disponível nesta fase documental.
- Fazer revisão de segurança de Sofia por pelo menos dois dias úteis antes do deploy.
- Executar checklist de aceite com produto, pedidos, plataforma e clientes envolvidos, registrando pendências de rate limiting e escala para a próxima decisão.

---

## Documentos relacionados

- [RFC — Webhooks outbound](RFC.md)
- [FDD — detalhamento técnico](FDD.md)
- [ADRs de decisões arquiteturais](adrs/)
