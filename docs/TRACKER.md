# Tracker de Rastreabilidade

Mapeamento de cada item registrado nos documentos (`docs/PRD.md`, `docs/RFC.md`, `docs/FDD.md`, `docs/adrs/*.md`) à sua origem: a transcrição da reunião técnica (`TRANSCRICAO.md`) ou o código-fonte existente.

## PRD (`docs/PRD.md`)

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| PRD-FR-01 | docs/PRD.md | Requisito Funcional | Cadastro de webhook (URL HTTPS, statusFilter), secret gerada e devolvida na criação | TRANSCRICAO | [09:31] Marcos |
| PRD-FR-02 | docs/PRD.md | Requisito Funcional | Edição de webhook (PATCH) | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-03 | docs/PRD.md | Requisito Funcional | Remoção de webhook (DELETE) | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-04 | docs/PRD.md | Requisito Funcional | Listagem de webhooks por customer (GET) | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-05 | docs/PRD.md | Requisito Funcional | Rotação de secret com grace period de 24h | TRANSCRICAO | [09:21] Sofia |
| PRD-FR-06 | docs/PRD.md | Requisito Funcional | Histórico das últimas 100 entregas do webhook | TRANSCRICAO | [09:34] Marcos |
| PRD-FR-07 | docs/PRD.md | Requisito Funcional | Filtro de eventos por status configurável por webhook | TRANSCRICAO | [09:33] Marcos |
| PRD-FR-08 | docs/PRD.md | Requisito Funcional | Emissão do evento deve ser consistente com a transação de status (sem caso de status mudar e evento não sair) | TRANSCRICAO | [09:40] Bruno |
| PRD-FR-09 | docs/PRD.md | Requisito Funcional | Assinatura HMAC-SHA256 do payload para autenticidade | TRANSCRICAO | [09:20] Sofia |
| PRD-FR-10 | docs/PRD.md | Requisito Funcional | Retry automático com backoff antes de falha permanente | TRANSCRICAO | [09:15] Diego |
| PRD-FR-11 | docs/PRD.md | Requisito Funcional | Identificador único de evento (event_id) para dedup at-least-once | TRANSCRICAO | [09:25] Diego |
| PRD-FR-12 | docs/PRD.md | Requisito Funcional | Replay manual de DLQ restrito a role ADMIN, com auditoria | TRANSCRICAO | [09:36] Sofia |
| PRD-NFR-01 | docs/PRD.md | Requisito Não Funcional | Latência de entrega inicial abaixo de 10 segundos | TRANSCRICAO | [09:02] Marcos |
| PRD-NFR-02 | docs/PRD.md | Requisito Não Funcional | URL de webhook deve ser HTTPS | TRANSCRICAO | [09:23] Sofia |
| PRD-NFR-03 | docs/PRD.md | Requisito Não Funcional | Payload acima de 64KB deve ser rejeitado, não truncado | TRANSCRICAO | [09:24] Diego |
| PRD-NFR-04 | docs/PRD.md | Requisito Não Funcional | Secret exclusiva por endpoint (não global) | TRANSCRICAO | [09:21] Sofia |
| PRD-NFR-05 | docs/PRD.md | Requisito Não Funcional | Retry deve sobreviver a até ~15h de indisponibilidade do cliente | TRANSCRICAO | [09:17] Diego |
| PRD-NFR-06 | docs/PRD.md | Requisito Não Funcional | Solução não deve exigir infraestrutura nova além de MySQL/Prisma | TRANSCRICAO | [09:07] Diego |
| PRD-NFR-07 | docs/PRD.md | Requisito Não Funcional | Timeout de chamada HTTP do worker: 10 segundos | TRANSCRICAO | [09:42] Diego |
| PRD-OBJ-01 | docs/PRD.md | Objetivo/Métrica | Eliminar polling manual: 100% das mudanças assinadas geram tentativa de entrega | TRANSCRICAO | [09:00] Marcos |
| PRD-OBJ-02 | docs/PRD.md | Objetivo/Métrica | Meta de latência abaixo de 10s definida pelos clientes | TRANSCRICAO | [09:02] Marcos |
| PRD-OBJ-03 | docs/PRD.md | Objetivo/Métrica | Estimativa de entrega em 3 sprints, incluindo revisão de segurança | TRANSCRICAO | [09:46] Larissa |
| PRD-OBJ-04 | docs/PRD.md | Objetivo/Métrica | Cobertura de retry para ~15h de indisponibilidade antes de falha permanente | TRANSCRICAO | [09:17] Diego |
| PRD-SCOPE-OUT-01 | docs/PRD.md | Restrição (fora de escopo) | Notificação por e-mail em falhas recorrentes adiada para fase futura | TRANSCRICAO | [09:37] Larissa |
| PRD-SCOPE-OUT-02 | docs/PRD.md | Restrição (fora de escopo) | Rate limiting de envio não implementado agora; observar e decidir depois | TRANSCRICAO | [09:39] Larissa |
| PRD-SCOPE-OUT-03 | docs/PRD.md | Restrição (fora de escopo) | Dashboard visual para o cliente fora de escopo | TRANSCRICAO | [09:40] Larissa |
| PRD-SCOPE-OUT-04 | docs/PRD.md | Restrição (fora de escopo) | Arquivamento automático de eventos entregues após 30 dias fora de escopo | TRANSCRICAO | [09:08] Diego |
| PRD-SCOPE-OUT-05 | docs/PRD.md | Restrição (fora de escopo) | Múltiplos workers com ordering global garantido fica para o futuro | TRANSCRICAO | [09:13] Diego |
| PRD-RISK-01 | docs/PRD.md | Risco | Cliente Atlas Comercial pode migrar para concorrente por atraso | TRANSCRICAO | [09:00] Marcos |
| PRD-RISK-02 | docs/PRD.md | Risco | Vazamento de secret compromete integração de um cliente (incidente já ocorrido) | TRANSCRICAO | [09:22] Diego |
| PRD-RISK-03 | docs/PRD.md | Risco | Payload de evento excessivamente grande degrada envio | TRANSCRICAO | [09:24] Diego |
| PRD-DEP-01 | docs/PRD.md | Dependência | Método `changeStatus` do módulo de pedidos é a origem única dos eventos | CODIGO | src/modules/orders/order.service.ts |
| PRD-DEP-02 | docs/PRD.md | Dependência | Infraestrutura MySQL/Prisma já provisionada, sem novo componente | CODIGO | prisma/schema.prisma |
| PRD-DEP-03 | docs/PRD.md | Dependência | Revisão de segurança dedicada da Sofia antes do deploy (2 dias úteis) | TRANSCRICAO | [09:46] Sofia |
| PRD-DEP-04 | docs/PRD.md | Dependência | Documentação do comportamento at-least-once no portal do desenvolvedor | TRANSCRICAO | [09:26] Marcos |

## RFC (`docs/RFC.md`)

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| RFC-CTX-01 | docs/RFC.md | Contexto | Clientes B2B pedem notificação em tempo real em vez de polling em GET /orders | TRANSCRICAO | [09:00] Marcos |
| RFC-PROP-01 | docs/RFC.md | Decisão | Emissão assíncrona via outbox dentro da transação de changeStatus | TRANSCRICAO | [09:06] Diego |
| RFC-PROP-02 | docs/RFC.md | Decisão | Worker dedicado em processo separado | TRANSCRICAO | [09:11] Diego |
| RFC-PROP-03 | docs/RFC.md | Decisão | Resiliência via retry exponencial e DLQ | TRANSCRICAO | [09:15] Diego |
| RFC-PROP-04 | docs/RFC.md | Decisão | Segurança via HMAC-SHA256 e secret por endpoint | TRANSCRICAO | [09:20] Sofia |
| RFC-PROP-05 | docs/RFC.md | Decisão | Contrato de entrega at-least-once com X-Event-Id | TRANSCRICAO | [09:24] Diego |
| RFC-ALT-01 | docs/RFC.md | Trade-off | Disparo síncrono dentro de changeStatus descartado (risco de travar outros pedidos) | TRANSCRICAO | [09:04] Bruno |
| RFC-ALT-02 | docs/RFC.md | Trade-off | Fila dedicada (Redis Streams) descartada por overengineering para time pequeno | TRANSCRICAO | [09:07] Larissa |
| RFC-ALT-03 | docs/RFC.md | Trade-off | Retry agressivo de 3 tentativas descartado (mataria eventos em manutenções longas) | TRANSCRICAO | [09:16] Diego |
| RFC-OPEN-01 | docs/RFC.md | Restrição (questão em aberto) | Rate limiting de envio por cliente não decidido, fica em observação | TRANSCRICAO | [09:38] Diego |
| RFC-OPEN-02 | docs/RFC.md | Restrição (questão em aberto) | Notificação de falha recorrente por e-mail adiada | TRANSCRICAO | [09:37] Larissa |
| RFC-OPEN-03 | docs/RFC.md | Restrição (questão em aberto) | Escala do worker para múltiplas instâncias não resolvida nesta fase | TRANSCRICAO | [09:13] Diego |
| RFC-IMPACT-01 | docs/RFC.md | Risco | Exposição de dados de pedidos a sistemas fora da infraestrutura da empresa | TRANSCRICAO | [09:19] Sofia |
| RFC-IMPACT-02 | docs/RFC.md | Risco | Novo processo (worker) a operar e monitorar, análogo ao entry-point da API | CODIGO | src/server.ts |
| RFC-IMPACT-03 | docs/RFC.md | Risco | Prazo de 3 sprints para atender compromisso com a Atlas | TRANSCRICAO | [09:46] Larissa |

## FDD (`docs/FDD.md`)

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| FDD-FLOW-01 | docs/FDD.md | Requisito Funcional | Inserção do evento na outbox dentro da transação de changeStatus | CODIGO | src/modules/orders/order.service.ts |
| FDD-FLOW-02 | docs/FDD.md | Requisito Funcional | Worker processa eventos pendentes em polling de 2s | TRANSCRICAO | [09:09] Diego |
| FDD-FLOW-03 | docs/FDD.md | Requisito Funcional | Retry com backoff exponencial recalcula next_attempt_at | TRANSCRICAO | [09:17] Diego |
| FDD-FLOW-04 | docs/FDD.md | Requisito Funcional | Evento esgotado move para tabela de DLQ separada | TRANSCRICAO | [09:18] Diego |
| FDD-CONTRATO-01 | docs/FDD.md | Contrato | POST /api/v1/webhooks — cadastro | TRANSCRICAO | [09:31] Marcos |
| FDD-CONTRATO-02 | docs/FDD.md | Contrato | GET /api/v1/webhooks — listagem por customer | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-03 | docs/FDD.md | Contrato | PATCH /api/v1/webhooks/:id — edição | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-04 | docs/FDD.md | Contrato | DELETE /api/v1/webhooks/:id — remoção | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-05 | docs/FDD.md | Contrato | POST /api/v1/webhooks/:id/rotate-secret | TRANSCRICAO | [09:21] Sofia |
| FDD-CONTRATO-06 | docs/FDD.md | Contrato | GET /api/v1/webhooks/:id/deliveries — histórico | TRANSCRICAO | [09:34] Marcos |
| FDD-CONTRATO-07 | docs/FDD.md | Contrato | POST /api/v1/admin/webhooks/dead-letter/:id/replay | TRANSCRICAO | [09:18] Diego |
| FDD-ERR-01 | docs/FDD.md | Padrão | Formato de erro `{ error: { code, message, details } }` reaproveitado | CODIGO | src/middlewares/error.middleware.ts |
| FDD-ERR-02 | docs/FDD.md | Requisito Funcional | WEBHOOK_INVALID_URL para URL não-HTTPS | TRANSCRICAO | [09:23] Sofia |
| FDD-ERR-03 | docs/FDD.md | Requisito Funcional | WEBHOOK_PAYLOAD_TOO_LARGE para payload acima de 64KB | TRANSCRICAO | [09:23] Sofia |
| FDD-ERR-04 | docs/FDD.md | Decisão | Prefixo WEBHOOK_ para todos os códigos de erro do módulo | TRANSCRICAO | [09:29] Larissa |
| FDD-ERR-05 | docs/FDD.md | Padrão | Classes de erro do módulo seguem padrão de InsufficientStockError/InvalidStatusTransitionError | CODIGO | src/shared/errors/http-errors.ts |
| FDD-RESIL-01 | docs/FDD.md | Requisito Não Funcional | Timeout de 10s por chamada HTTP do worker | TRANSCRICAO | [09:42] Diego |
| FDD-RESIL-02 | docs/FDD.md | Requisito Não Funcional | Intervalos de backoff: 1m/5m/30m/2h/12h | TRANSCRICAO | [09:17] Diego |
| FDD-RESIL-03 | docs/FDD.md | Requisito Funcional | Fallback: replay manual via endpoint admin após DLQ | TRANSCRICAO | [09:18] Diego |
| FDD-RESIL-04 | docs/FDD.md | Requisito Não Funcional | Limite de payload de 64KB | TRANSCRICAO | [09:24] Diego |
| FDD-RESIL-05 | docs/FDD.md | Decisão | Isolamento de processo entre API e worker | TRANSCRICAO | [09:11] Diego |
| FDD-OBS-01 | docs/FDD.md | Padrão | Logs estruturados via logger Pino já configurado | CODIGO | src/shared/logger/index.ts |
| FDD-OBS-02 | docs/FDD.md | Requisito Funcional | X-Event-Id como identificador de correlação/tracing | TRANSCRICAO | [09:25] Diego |
| FDD-OBS-03 | docs/FDD.md | Requisito Funcional | Log de auditoria de quem executou o replay de DLQ | TRANSCRICAO | [09:36] Sofia |
| FDD-MODEL-01 | docs/FDD.md | Decisão | Índices em status e created_at na tabela outbox | TRANSCRICAO | [09:07] Diego |
| FDD-MODEL-02 | docs/FDD.md | Decisão | IDs em UUID, seguindo padrão do restante do projeto | TRANSCRICAO | [09:51] Larissa |
| FDD-MODEL-03 | docs/FDD.md | Decisão | Payload armazenado como snapshot renderizado na inserção | TRANSCRICAO | [09:52] Larissa |
| FDD-MODEL-04 | docs/FDD.md | Decisão | Tabela de DLQ separada da outbox principal | TRANSCRICAO | [09:18] Diego |
| FDD-INTEG-01 | docs/FDD.md | Restrição | Extensão de changeStatus para chamar publishWebhookEvent(tx, ...) | CODIGO | src/modules/orders/order.service.ts |
| FDD-INTEG-02 | docs/FDD.md | Restrição | Novas classes de erro estendem AppError/http-errors.ts existentes | CODIGO | src/shared/errors/http-errors.ts |
| FDD-INTEG-03 | docs/FDD.md | Restrição | Endpoint de replay reaproveita requireRole('ADMIN') sem alterar o middleware | CODIGO | src/middlewares/auth.middleware.ts |
| FDD-INTEG-04 | docs/FDD.md | Restrição | Registro das novas rotas em buildApiRouter | CODIGO | src/routes/index.ts |
| FDD-INTEG-05 | docs/FDD.md | Restrição | Composição de repository/service/controller em buildControllers | CODIGO | src/app.ts |
| FDD-INTEG-06 | docs/FDD.md | Restrição | Novo script "worker" espelhando o script "dev" existente | CODIGO | package.json |
| FDD-INTEG-07 | docs/FDD.md | Restrição | Novos models Prisma seguem padrão de uuid/índices dos models existentes | CODIGO | prisma/schema.prisma |
| FDD-INTEG-08 | docs/FDD.md | Restrição | Worker reaproveita o logger Pino singleton, sem configuração própria | CODIGO | src/shared/logger/index.ts |

## ADRs (`docs/adrs/`)

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| ADR-001 | docs/adrs/ADR-001-outbox-pattern-no-mysql.md | Decisão | Padrão outbox no MySQL, inserção na mesma transação de changeStatus | TRANSCRICAO | [09:06] Diego |
| ADR-001-ALT | docs/adrs/ADR-001-outbox-pattern-no-mysql.md | Trade-off | Redis Streams descartado por exigir infraestrutura nova | TRANSCRICAO | [09:07] Larissa |
| ADR-001-CODE | docs/adrs/ADR-001-outbox-pattern-no-mysql.md | Restrição | Ponto de inserção dentro da transação existente | CODIGO | src/modules/orders/order.service.ts |
| ADR-002 | docs/adrs/ADR-002-worker-em-processo-separado-com-polling.md | Decisão | Worker roda como processo Node separado, polling de 2s | TRANSCRICAO | [09:11] Diego |
| ADR-002-ALT | docs/adrs/ADR-002-worker-em-processo-separado-com-polling.md | Trade-off | Trigger de banco descartado (MySQL não tem LISTEN/NOTIFY) | TRANSCRICAO | [09:09] Diego |
| ADR-002-CODE | docs/adrs/ADR-002-worker-em-processo-separado-com-polling.md | Restrição | Novo entry-point análogo a src/server.ts | CODIGO | src/server.ts |
| ADR-003 | docs/adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md | Decisão | 5 tentativas, backoff 1m/5m/30m/2h/12h, depois DLQ | TRANSCRICAO | [09:17] Diego |
| ADR-003-ALT | docs/adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md | Trade-off | 3 tentativas descartadas por matar eventos em manutenções longas | TRANSCRICAO | [09:16] Diego |
| ADR-004 | docs/adrs/ADR-004-autenticacao-hmac-sha256-com-secret-por-endpoint.md | Decisão | HMAC-SHA256 com secret por endpoint e rotação com grace period de 24h | TRANSCRICAO | [09:20] Sofia |
| ADR-004-ALT | docs/adrs/ADR-004-autenticacao-hmac-sha256-com-secret-por-endpoint.md | Trade-off | Secret global única descartada (blast radius de vazamento) | TRANSCRICAO | [09:21] Sofia |
| ADR-005 | docs/adrs/ADR-005-garantia-at-least-once-com-x-event-id.md | Decisão | Garantia at-least-once com dedup via X-Event-Id | TRANSCRICAO | [09:24] Diego |
| ADR-005-ALT | docs/adrs/ADR-005-garantia-at-least-once-com-x-event-id.md | Trade-off | Exactly-once descartado por complexidade de coordenação distribuída | TRANSCRICAO | [09:25] Diego |
| ADR-006 | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Decisão | Reuso máximo de AppError, Pino, error middleware e padrão modular | TRANSCRICAO | [09:30] Larissa |
| ADR-006-CODE-01 | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Restrição | Estrutura de módulo (controller/service/repository/routes/schemas) usada como referência | CODIGO | src/modules/customers/customer.controller.ts |
| ADR-006-CODE-02 | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Restrição | Middleware de validação Zod reaproveitado | CODIGO | src/middlewares/validate.middleware.ts |
| ADR-007 | docs/adrs/ADR-007-snapshot-do-payload-na-insercao-do-outbox.md | Decisão | Payload renderizado (snapshot) no momento da inserção na outbox | TRANSCRICAO | [09:52] Larissa |
| ADR-007-ALT | docs/adrs/ADR-007-snapshot-do-payload-na-insercao-do-outbox.md | Trade-off | Renderização tardia (só order_id) descartada por inconsistência com estado do pedido | TRANSCRICAO | [09:51] Bruno |
