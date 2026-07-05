# ADR-005: Garantia de entrega at-least-once com deduplicação via X-Event-Id

## Status

Aceito

## Contexto

Dado o modelo de retry (ADR-003) e a possibilidade de falhas de rede após o processamento bem-sucedido no lado do cliente (o cliente recebe e processa, mas a confirmação se perde), é possível que um mesmo evento seja entregue mais de uma vez. Diego colocou explicitamente que a plataforma vai garantir at-least-once, não exactly-once, e que o cliente precisa estar preparado para lidar com duplicatas (`[09:24] Diego`).

Sofia observou que isso transfere responsabilidade para o lado do cliente (`[09:25] Sofia`). Diego defendeu que esse é o padrão de mercado adotado por players como Stripe e GitHub, e que garantir exactly-once exigiria coordenação entre os dois lados, adicionando complexidade desproporcional ao ganho (`[09:25] Diego`).

## Decisão

Cada evento gerado recebe um `event_id` (UUID) no momento em que entra na outbox. Esse identificador é enviado em todo envio (incluindo reenvios) no header `X-Event-Id`. O cliente é responsável por deduplicar eventos recebidos mais de uma vez usando esse identificador do lado dele (`[09:25] Diego`). A plataforma documenta esse comportamento de forma destacada no portal do desenvolvedor para os clientes (`[09:26] Marcos`).

## Alternativas Consideradas

- **Garantia exactly-once**: descartada por exigir protocolos de coordenação distribuída entre plataforma e cliente (ex: two-phase commit ou acknowledgment transacional), com complexidade de implementação em ambos os lados desproporcional ao problema — a equipe estimou que at-least-once com dedup resolve "99% dos casos" (`[09:25] Diego`).
- **At-most-once (sem retry após primeira tentativa)**: implicitamente descartada — conflita diretamente com o modelo de retry já decidido (ADR-003), que existe justamente para tentar garantir que o cliente receba o evento mesmo após falhas transitórias.

## Consequências

**Positivas:**
- Simplicidade de implementação no lado da plataforma: não é necessário rastrear confirmações de processamento do cliente.
- Alinhado com o padrão adotado por provedores de referência (Stripe, GitHub), facilitando a integração para clientes já familiarizados com o modelo.
- `X-Event-Id` também serve de correlação para consulta de histórico de entregas (`GET /webhooks/:id/deliveries`).

**Negativas:**
- Transfere para o cliente a responsabilidade de implementar deduplicação — se ele não implementar, pode processar o mesmo evento de negócio mais de uma vez.
- Não elimina duplicatas na origem, apenas fornece o meio (event_id) para que o destinatário as trate.
