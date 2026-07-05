# RFC — Sistema de Webhooks de Notificação de Pedidos

## Metadados

| Campo | Valor |
| --- | --- |
| **Autor** | Larissa (Tech Lead) |
| **Status** | Em revisão |
| **Data** | Semana da reunião técnica de kickoff (ver `TRANSCRICAO.md`, quinta-feira, 09:00) |
| **Revisores** | Marcos (PM), Bruno (Eng. Pleno, Pedidos), Diego (Eng. Sênior, Plataforma), Sofia (Eng. Segurança) |

## Resumo executivo (TL;DR)

Vamos construir um sistema de webhooks outbound para notificar clientes B2B (Atlas Comercial, MaxDistribuição, Nova Cargo) sobre mudanças de status de seus pedidos, eliminando a necessidade de polling manual no `GET /orders`. A entrega será assíncrona via padrão Outbox no MySQL existente, processada por um worker em processo separado (polling de 2s), com retry exponencial e DLQ para falhas persistentes, autenticação HMAC-SHA256 por endpoint, e garantia de entrega at-least-once com deduplicação por `event_id`. O módulo reaproveita integralmente os padrões arquiteturais já estabelecidos no projeto (estrutura modular, `AppError`, logger Pino, error middleware, `requireRole`). Estimativa: 3 sprints, incluindo revisão de segurança da Sofia (`[09:46] Larissa`).

## Contexto e problema

Três clientes B2B formalizaram o pedido de notificação em tempo real de mudanças de status de pedidos, hoje resolvido via polling manual em `GET /orders`, que é lento e caro para eles de manter (`[09:00] Marcos`). O requisito de latência aceito pelos clientes é "abaixo de 10 segundos" (`[09:02] Marcos`). Há risco de perda de conta: a Atlas sinalizou possível migração para concorrente caso a entrega não ocorra até o fim do trimestre (`[09:00] Marcos`).

A aplicação hoje (`src/`) não possui nenhum mecanismo de eventos, filas ou notificação externa — esse vácuo é o que esta feature preenche. O ponto de integração central é `OrderService.changeStatus` (`src/modules/orders/order.service.ts:126-179`), que já executa, em uma transação Prisma, a atualização de `orders`, a inserção em `order_status_history` e o ajuste de `stock_quantity`.

## Proposta técnica

A proposta tem cinco pilares, cada um formalizado em um ADR dedicado (ver seção "Decisões relacionadas"):

1. **Emissão assíncrona via Outbox em MySQL**: dentro da mesma transação que já roda em `changeStatus`, inserimos um evento na tabela `webhook_outbox`. Se a transação falhar, o evento nunca existiu; se ela commitar, o evento está garantido. Nenhum HTTP síncrono acontece dentro da transação de negócio.
2. **Worker dedicado em polling**: um processo Node separado (`src/worker.ts`, novo script `npm run worker`) varre a outbox a cada 2 segundos, processando eventos pendentes em ordem e disparando as chamadas HTTP para os endpoints cadastrados.
3. **Resiliência via retry + DLQ**: falhas de entrega acionam backoff exponencial (5 tentativas, ~15h de janela total); esgotadas as tentativas, o evento vai para uma tabela `webhook_dead_letter`, reprocessável manualmente por um endpoint administrativo.
4. **Segurança por HMAC e secret por endpoint**: cada endpoint de webhook tem secret própria, rotacionável com grace period de 24h; o payload é assinado com HMAC-SHA256 (`X-Signature`), e a URL cadastrada deve ser HTTPS.
5. **Contrato de entrega at-least-once**: cada evento carrega um `X-Event-Id` único (UUID), permitindo que o cliente deduplique entregas repetidas do lado dele — modelo adotado por provedores de referência como Stripe e GitHub.

O módulo novo (`src/modules/webhooks/`) segue a mesma estrutura de controller/service/repository/routes/schemas dos módulos existentes, reaproveitando `AppError`, o logger Pino, o error middleware central e o middleware `requireRole` para proteger o endpoint administrativo de replay da DLQ. O detalhamento de contratos de API, matriz de erros, fluxos passo a passo e observabilidade está no FDD (`docs/FDD.md`).

## Alternativas consideradas

- **Disparo síncrono dentro de `OrderService.changeStatus`**: descartada porque acoplaria a disponibilidade/latência de sistemas de clientes externos à transação crítica de mudança de status — um cliente lento ou fora do ar travaria a atualização de outros pedidos, sem estratégia de rollback coerente (`[09:04] Bruno`). *Trade-off:* simplicidade de implementação vs. risco real de indisponibilidade em cascata.
- **Fila dedicada (ex: Redis Streams)**: descartada por exigir subir e operar infraestrutura nova (ex: Redis Cluster) para um time pequeno, sendo considerada overengineering frente ao volume e à criticidade do problema atual — o MySQL existente já resolve via outbox (`[09:07] Diego`, `[09:07] Larissa`). *Trade-off:* uma fila dedicada ofereceria melhor throughput e reatividade nativa, mas ao custo de operação de mais um componente de infraestrutura.
- **Retry agressivo de 3 tentativas**: descartada em favor de 5 tentativas — 3 tentativas em uma janela curta (~30 min) mataria eventos durante janelas de manutenção planejada legítimas já observadas em clientes reais (`[09:16] Diego`). *Trade-off:* menos tentativas reduzem carga operacional do worker, mas aumentam falsos negativos de entrega.

## Questões em aberto

- **Rate limiting de envio por cliente**: se um cliente tiver dezenas de pedidos mudando de status no mesmo minuto, a plataforma pode enviar uma rajada de chamadas HTTP para o endpoint dele. A equipe decidiu não implementar isso nesta fase, apenas observar e decidir depois se virar problema real (`[09:38]-[09:39] Diego, Larissa`).
- **Notificação proativa de endpoint com falha recorrente**: a ideia de avisar o cliente por e-mail quando o webhook dele falha repetidamente (ex: 3 falhas seguidas) foi explicitamente colocada fora de escopo desta fase, ficando como possível fase futura após medição de impacto (`[09:37]-[09:38] Larissa, Marcos`).
- **Escala do worker para múltiplas instâncias**: hoje a proposta assume single-worker para preservar ordenação por `order_id`. Particionamento por `order_id` ou lock pessimista para permitir múltiplos workers foi mencionado como problema a resolver no futuro, não nesta entrega (`[09:13] Diego`).

## Impacto e riscos

- **Impacto no `OrderService`**: `changeStatus` passa a ter uma responsabilidade adicional dentro da transação (inserir na outbox). Qualquer falha nessa inserção deve propagar rollback — comportamento a validar cuidadosamente em teste de integração.
- **Risco de segurança**: exposição de dados de pedidos para fora da infraestrutura da empresa é meta risco central desta feature; mitigado por HMAC-SHA256, secret por endpoint, TLS obrigatório e revisão de segurança dedicada da Sofia antes do deploy (`[09:46] Sofia` reservou ao menos dois dias úteis para essa revisão).
- **Risco operacional**: novo processo (`worker`) a monitorar e implantar; requer operação e observabilidade equivalentes às da API (ver FDD, seção Observabilidade).
- **Risco de prazo**: estimativa de 3 sprints, incluindo a revisão de segurança, para atender o compromisso comunicado à Atlas (fim do trimestre) (`[09:45]-[09:46] Marcos, Larissa`).

## Decisões relacionadas

- [ADR-001 — Padrão Outbox no MySQL](adrs/ADR-001-outbox-pattern-no-mysql.md)
- [ADR-002 — Worker em processo separado com polling](adrs/ADR-002-worker-em-processo-separado-com-polling.md)
- [ADR-003 — Retry com backoff exponencial e DLQ](adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md)
- [ADR-004 — HMAC-SHA256 com secret por endpoint](adrs/ADR-004-autenticacao-hmac-sha256-com-secret-por-endpoint.md)
- [ADR-005 — Garantia at-least-once com X-Event-Id](adrs/ADR-005-garantia-at-least-once-com-x-event-id.md)
- [ADR-006 — Reuso dos padrões existentes do projeto](adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md)
- [ADR-007 — Snapshot do payload na inserção do outbox](adrs/ADR-007-snapshot-do-payload-na-insercao-do-outbox.md)
