# ADR-004: HMAC-SHA256 e segredo por endpoint

## Status

Aceito para o design. A implementação da funcionalidade não faz parte desta entrega.

## Contexto

Os webhooks transportam dados de pedidos para endpoints fora da infraestrutura da plataforma. Sofia destacou que o cliente precisa verificar a origem e detectar adulteração do corpo. A reunião definiu HMAC-SHA256 sobre o corpo, com a assinatura em `X-Signature`, entre 09:19 e 09:22.

Um segredo global ampliaria o impacto de qualquer vazamento. Por isso, cada endpoint de webhook deve ter um segredo próprio. A rotação precisa permitir migração sem interrupção: o segredo antigo permanece válido por 24 horas. A URL também deve ser HTTPS; HTTP deve ser recusado. `X-Timestamp` acompanha o envio para que o cliente possa avaliar replay, se desejar.

## Decisão

Assinar o corpo do request com HMAC-SHA256 usando um segredo exclusivo do endpoint e enviar o resultado em `X-Signature`. Exigir HTTPS no cadastro e no envio. Gerar e guardar um segredo por endpoint, com rotação que mantém o segredo anterior válido por um grace period de 24 horas.

Enviar também `X-Timestamp` com o timestamp do envio. A codificação exata da assinatura, a representação canônica e os mapeamentos de erro que não foram fechados na reunião devem ser descritos como contrato proposto no FDD; este ADR não inventa esses detalhes.

## Alternativas Consideradas

| Alternativa | Motivo da rejeição | Trade-off |
|---|---|---|
| Um segredo global para toda a plataforma | Sofia rejeitou essa direção porque o vazamento de um segredo comprometeria todos os endpoints. | Simplificaria configuração e rotação, mas aumentaria drasticamente o raio de impacto de uma exposição. |
| Apenas HTTPS, sem assinatura do corpo | HTTPS protege o transporte, mas não fornece ao consumidor a verificação de integridade e origem no contrato do webhook. | Teria menos complexidade de chave, mas não atenderia a necessidade de o cliente validar o corpo recebido. |

## Consequências

### Positivas

- O consumidor consegue verificar a integridade do payload com uma técnica amplamente disponível.
- O comprometimento de um endpoint não exige compartilhar o mesmo segredo com os demais.
- A rotação com 24 horas de sobreposição reduz a interrupção da migração.
- HTTPS evita cadastrar ou enviar para uma URL HTTP.

### Negativas

- Segredos precisam ser gerados, armazenados, rotacionados e excluídos de logs.
- Durante 24 horas, dois segredos válidos precisam ser tratados para o mesmo endpoint.
- Clientes precisam implementar a validação HMAC e considerar `X-Timestamp` conforme sua política.
- A reunião não fechou canonicalização, formato textual da assinatura ou mecanismo de rate limiting; esses pontos não podem ser inventados neste ADR.

## Relações e referências

- O identificador do evento e a semântica de duplicatas estão no [ADR-005](ADR-005-at-least-once-e-x-event-id.md).
- Os padrões de autenticação, erros e logging existentes estão no [ADR-006](ADR-006-reuso-dos-padroes-do-projeto.md).
- A proposta de alto nível será consolidada no [RFC](../RFC.md) e os headers/contratos no [FDD](../FDD.md).
- Evidências primárias: [TRANSCRICAO.md](../../TRANSCRICAO.md), `[09:19–09:24]` e `[09:44–09:45]`.
