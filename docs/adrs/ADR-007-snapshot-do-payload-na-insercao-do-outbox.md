# ADR-007: Payload do evento é renderizado (snapshot) no momento da inserção na outbox

## Status

Aceito

## Contexto

Ao modelar a tabela `webhook_outbox`, Bruno levantou uma dúvida de design: o evento deveria armazenar apenas o `order_id` e montar o payload de envio na hora da entrega (leitura tardia), ou já armazenar o payload renderizado no momento em que o evento é inserido na outbox (`[09:51] Bruno`)?

Essa dúvida foi levantada após o encerramento formal da reunião, em uma conversa rápida entre Larissa, Diego e Bruno.

## Decisão

O payload do evento é renderizado (montado) no momento da inserção na outbox, não no momento do envio. Larissa justificou que, se o pedido mudar de estado novamente antes do worker processar o evento mais antigo, o evento entregue deve refletir o estado do pedido no momento exato da transição que o originou — não o estado mais recente do pedido (`[09:52] Larissa`). Diego e Bruno concordaram com o modelo de snapshot na inserção (`[09:52] Diego`, `[09:52] Bruno`).

Isso é coerente com a decisão de payload enxuto (`event_id`, `event_type`, `timestamp`, `order_id`, `order_number`, `from_status`, `to_status`, `customer_id` e campos básicos como `total_cents`), sem incluir a lista de itens do pedido (`[09:43] Diego`).

## Alternativas Consideradas

- **Renderização tardia (armazenar apenas `order_id` e montar o payload no momento do envio)**: rejeitada porque, se o pedido sofresse mudanças adicionais entre a inserção do evento e o efetivo envio pelo worker, o payload enviado refletiria um estado diferente daquele que gerou o evento originalmente — criando inconsistência entre o `to_status` do evento e o estado real do pedido no momento da transição (`[09:52] Larissa`).

## Consequências

**Positivas:**
- Garante que o payload entregue ao cliente sempre reflete fielmente o estado do pedido no exato momento da transição de status que originou o evento, mesmo em caso de retries horas depois.
- Evita uma consulta adicional ao pedido no momento do envio, simplificando o worker (ele apenas lê e envia o payload já pronto).

**Negativas:**
- Aumenta o tamanho médio das linhas da tabela `webhook_outbox`, pois o payload completo é armazenado por evento (mitigado pelo limite de 64KB por payload).
- Se o formato do payload mudar no futuro (nova versão de contrato), eventos antigos ainda pendentes na outbox continuam com o formato antigo até serem processados — versionamento de payload precisa ser considerado à parte.
