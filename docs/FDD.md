# FDD — Feature Design Document: Sistema de Webhooks de Notificação de Pedidos

## Contexto e motivação técnica

O RFC (`docs/RFC.md`) definiu a abordagem de arquitetura: outbox no MySQL, worker dedicado, retry com DLQ, HMAC-SHA256 e at-least-once via `X-Event-Id`. Este documento detalha como implementar essa proposta: modelo de dados, fluxos passo a passo, contratos de API completos, matriz de erros e pontos exatos de integração com o código existente do OMS.

O gatilho de todo o fluxo é a transição de status de um pedido, hoje implementada em `OrderService.changeStatus` (`src/modules/orders/order.service.ts:126-179`), a única rota do sistema que muda `orders.status`.

## Objetivos técnicos

- Garantir que todo evento de mudança de status gere, de forma atômica, uma linha na outbox — sem casos possíveis de status mudar e o evento não ser registrado (`[09:40] Bruno`).
- Entregar eventos aos endpoints dos clientes com latência de pior caso dominada pelo intervalo de polling do worker (2s) mais o tempo de resposta do endpoint remoto.
- Resistir a indisponibilidades transitórias de clientes por até ~15h via retry, sem intervenção manual.
- Autenticar entregas de forma verificável pelo cliente (HMAC-SHA256) sem expor uma secret compartilhada entre clientes distintos.
- Não introduzir nenhuma dependência de infraestrutura nova além do MySQL/Prisma já existente.

## Escopo e exclusões

**Dentro do escopo:**
- CRUD de configuração de webhook (URL, secret, filtro de status, estado ativo) por `customer_id`.
- Rotação de secret com grace period de 24h.
- Emissão de evento na outbox dentro da transação de `changeStatus`, com filtragem por status assinado no cadastro do webhook.
- Worker de processamento em polling, com retry exponencial e DLQ.
- Endpoint de histórico de entregas (`GET /webhooks/:id/deliveries`).
- Endpoint administrativo de replay manual de itens da DLQ, restrito a `ADMIN`.

**Fora do escopo (ver PRD para detalhamento):**
- Notificação por e-mail em caso de falhas recorrentes de um webhook (`[09:37]-[09:38] Larissa, Marcos`).
- Rate limiting de envio por cliente (`[09:38]-[09:39] Diego, Larissa`).
- Painel visual (dashboard) para o cliente gerenciar webhooks (`[09:39]-[09:40] Marcos, Larissa`).
- Arquivamento automático de eventos entregues após 30 dias (`[09:07] Diego`).
- Suporte a múltiplos workers em paralelo com garantia de ordering global (`[09:12]-[09:13] Diego, Bruno`).

## Modelo de dados (novo)

Quatro tabelas novas, seguindo o padrão de UUID (`db.Char(36)`, `@default(uuid())`) já usado em todo o `prisma/schema.prisma`:

- **`webhook_endpoints`**: `id`, `customer_id`, `url`, `secret`, `previous_secret` (nullable), `previous_secret_expires_at` (nullable), `status_filter` (lista de `OrderStatus`), `active`, `created_at`, `updated_at`.
- **`webhook_outbox`**: `id`, `event_id` (UUID, valor de `X-Event-Id`), `webhook_endpoint_id`, `order_id`, `payload` (JSON, snapshot renderizado — ver ADR-007), `status` (`PENDING`, `PROCESSING`, `FAILED`, `DELIVERED`), `attempts`, `next_attempt_at`, `created_at`. Índices em `status` e `created_at` (`[09:07] Diego`).
- **`webhook_dead_letter`**: `id`, `webhook_outbox_id` (referência ao evento original), `payload`, `failure_reason`, `attempts`, `created_at`.
- **`webhook_deliveries`**: `id`, `webhook_endpoint_id`, `event_id`, `success`, `response_status`, `response_time_ms`, `attempted_at` — histórico consultado por `GET /webhooks/:id/deliveries`.

## Fluxos detalhados

### 1. Criação do evento na outbox (dentro de `changeStatus`)

1. `OrderService.changeStatus` inicia a transação Prisma (já existente).
2. Após validar a transição (`canTransition`) e aplicar débito/reposição de estoque, **antes do commit**, o service chama uma função `publishWebhookEvent(tx, order, fromStatus, toStatus)` (`[09:41] Bruno`, `[09:41] Diego`).
3. `publishWebhookEvent` busca, dentro da mesma `tx`, os `webhook_endpoints` ativos do `customer_id` do pedido cujo `status_filter` contenha `toStatus`. Se nenhum endpoint atender ao filtro, nenhuma linha é inserida (filtragem na inserção, não no envio — `[09:34] Bruno`, `[09:34] Diego`).
4. Para cada endpoint correspondente, insere uma linha em `webhook_outbox` com `event_id` novo (UUID), `status = PENDING`, e o `payload` já renderizado (snapshot do estado do pedido no momento da transição — ADR-007).
5. Se a inserção falhar por qualquer motivo, a exceção propaga e toda a transação (incluindo a mudança de status) sofre rollback — não existe caminho onde o status muda sem o evento ser gravado.
6. A transação commita normalmente; o worker passa a enxergar o(s) evento(s) pendente(s) no próximo ciclo de polling.

### 2. Processamento pelo worker

1. `src/worker.ts` inicia um loop com intervalo de 2 segundos (`[09:09] Diego`).
2. A cada ciclo, busca um lote pequeno de eventos com `status = PENDING` e `next_attempt_at <= now()`, ordenados por `created_at` (ordering por `order_id`/inserção, não global — ADR-002).
3. Marca os eventos selecionados como `PROCESSING` (evita double-processing caso o ciclo seguinte comece antes do atual terminar).
4. Para cada evento, monta a requisição HTTP: `POST` para a `url` cadastrada, headers `X-Event-Id`, `X-Signature` (HMAC-SHA256 do corpo com a `secret` do endpoint), `X-Timestamp`, `X-Webhook-Id` (`[09:44]-[09:45] Diego, Sofia`), `Content-Type: application/json`. Timeout de 10 segundos (`[09:42] Diego`).
5. Resposta HTTP `2xx` dentro do timeout: marca o evento como `DELIVERED`, grava linha de sucesso em `webhook_deliveries`.
6. Resposta não-`2xx`, timeout ou erro de rede: segue para o fluxo de retry.

### 3. Retry com backoff exponencial

1. Em caso de falha, incrementa `attempts` e calcula `next_attempt_at` conforme a tabela de backoff: 1min, 5min, 30min, 2h, 12h (para as tentativas 1 a 5) (`[09:17] Diego`).
2. Evento volta para `status = PENDING` com o novo `next_attempt_at`; o worker só volta a selecioná-lo quando esse horário chegar.
3. Grava linha de falha em `webhook_deliveries` (com `response_status`/motivo, quando disponível).
4. Se `attempts` já atingiu 5 e a tentativa atual também falhou, o evento segue para o fluxo de DLQ.

### 4. Dead Letter Queue (DLQ)

1. Ao esgotar as 5 tentativas, o evento é movido para `webhook_dead_letter` (payload, motivo da última falha, número de tentativas, timestamp) e removido/marcado como `FAILED` na outbox (`[09:18] Diego`).
2. Reprocessamento é manual, via `POST /admin/webhooks/dead-letter/:id/replay`, restrito a `ADMIN` (`[09:18] Diego`, `[09:35]-[09:36] Sofia`). O endpoint recria uma linha em `webhook_outbox` com `status = PENDING`, `attempts = 0`, e loga o `user.id` de quem executou o replay para auditoria (`[09:36] Sofia`).

## Contratos públicos

Todos os endpoints (exceto o de replay administrativo) exigem autenticação via `authenticate` (JWT), qualquer role (`[09:36]-[09:37] Sofia, Marcos`). Prefixo base: `/api/v1/webhooks`.

### `POST /api/v1/webhooks` — cadastrar webhook

Cria um endpoint de webhook para um customer. A secret é gerada pela plataforma e devolvida **apenas nesta resposta** (`[09:31] Marcos`).

Request:
```json
{
  "customerId": "8f14e45f-ceea-467e-8e3c-1f1c7c4b6a10",
  "url": "https://integrations.atlascomercial.com/webhooks/orders",
  "statusFilter": ["SHIPPED", "DELIVERED"]
}
```

Response `201 Created`:
```json
{
  "id": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "customerId": "8f14e45f-ceea-467e-8e3c-1f1c7c4b6a10",
  "url": "https://integrations.atlascomercial.com/webhooks/orders",
  "statusFilter": ["SHIPPED", "DELIVERED"],
  "secret": "whsec_8f3a1c2e9b7d4f6a0e5c1b8d3f2a7e6c",
  "active": true,
  "createdAt": "2026-07-05T13:20:00.000Z"
}
```

Erros possíveis: `WEBHOOK_INVALID_URL` (400), `WEBHOOK_CUSTOMER_NOT_FOUND` (404), `VALIDATION_ERROR` (400, filtro de status vazio ou inválido).

### `GET /api/v1/webhooks?customerId=...` — listar webhooks de um customer

Response `200 OK`:
```json
{
  "data": [
    {
      "id": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
      "customerId": "8f14e45f-ceea-467e-8e3c-1f1c7c4b6a10",
      "url": "https://integrations.atlascomercial.com/webhooks/orders",
      "statusFilter": ["SHIPPED", "DELIVERED"],
      "active": true,
      "createdAt": "2026-07-05T13:20:00.000Z"
    }
  ],
  "pagination": { "page": 1, "pageSize": 20, "total": 1, "totalPages": 1 }
}
```
Nota: a `secret` nunca é retornada em listagens, apenas na criação e na rotação. Erros possíveis: `WEBHOOK_CUSTOMER_NOT_FOUND` (404).

### `PATCH /api/v1/webhooks/:id` — editar webhook

Request (campos parciais, mesmo padrão de `updateCustomerSchema`):
```json
{
  "statusFilter": ["PAID", "SHIPPED", "DELIVERED"],
  "active": true
}
```

Response `200 OK`:
```json
{
  "id": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "customerId": "8f14e45f-ceea-467e-8e3c-1f1c7c4b6a10",
  "url": "https://integrations.atlascomercial.com/webhooks/orders",
  "statusFilter": ["PAID", "SHIPPED", "DELIVERED"],
  "active": true,
  "updatedAt": "2026-07-05T14:00:00.000Z"
}
```
Erros possíveis: `WEBHOOK_NOT_FOUND` (404), `WEBHOOK_INVALID_URL` (400), `VALIDATION_ERROR` (400).

### `DELETE /api/v1/webhooks/:id` — remover webhook

Response `204 No Content` (sem corpo). Erros possíveis: `WEBHOOK_NOT_FOUND` (404).

### `POST /api/v1/webhooks/:id/rotate-secret` — rotacionar secret

A secret atual passa a `previous_secret`, válida por 24h em paralelo com a nova (`[09:21] Sofia`).

Response `200 OK`:
```json
{
  "id": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "secret": "whsec_1a2b3c4d5e6f7a8b9c0d1e2f3a4b5c6d",
  "previousSecretExpiresAt": "2026-07-06T14:00:00.000Z"
}
```
Erros possíveis: `WEBHOOK_NOT_FOUND` (404).

### `GET /api/v1/webhooks/:id/deliveries` — histórico de entregas

Retorna os últimos 100 envios (sucesso/falha, tempo de resposta) (`[09:34]-[09:35] Marcos`).

Response `200 OK`:
```json
{
  "data": [
    {
      "eventId": "b3e1c9a0-6f2d-4a1e-9c3b-7d8e9f0a1b2c",
      "success": true,
      "responseStatus": 200,
      "responseTimeMs": 184,
      "attemptedAt": "2026-07-05T14:02:03.000Z"
    },
    {
      "eventId": "a1b2c3d4-5e6f-7a8b-9c0d-1e2f3a4b5c6d",
      "success": false,
      "responseStatus": null,
      "responseTimeMs": 10000,
      "attemptedAt": "2026-07-05T13:58:00.000Z"
    }
  ],
  "pagination": { "page": 1, "pageSize": 100, "total": 2, "totalPages": 1 }
}
```
Erros possíveis: `WEBHOOK_NOT_FOUND` (404).

### `POST /api/v1/admin/webhooks/dead-letter/:id/replay` — reprocessar item da DLQ (ADMIN)

Exige role `ADMIN` via `requireRole('ADMIN')` (`[09:35]-[09:36] Sofia, Larissa`).

Response `200 OK`:
```json
{
  "outboxEventId": "d4e5f6a7-8b9c-0d1e-2f3a-4b5c6d7e8f9a",
  "status": "PENDING",
  "requeuedAt": "2026-07-05T15:10:00.000Z",
  "requeuedBy": "7c9e6679-7425-40de-944b-e07fc1f90ae7"
}
```
Erros possíveis: `WEBHOOK_DEAD_LETTER_NOT_FOUND` (404), `FORBIDDEN` (403, role diferente de `ADMIN`).

### Payload do evento entregue ao cliente (não é um endpoint da nossa API — é o corpo do `POST` que o worker envia ao cliente)

```json
{
  "event_id": "b3e1c9a0-6f2d-4a1e-9c3b-7d8e9f0a1b2c",
  "event_type": "order.status_changed",
  "timestamp": "2026-07-05T14:02:00.000Z",
  "order_id": "9d8c7b6a-5f4e-3d2c-1b0a-9f8e7d6c5b4a",
  "order_number": "ORD-000123",
  "from_status": "PROCESSING",
  "to_status": "SHIPPED",
  "customer_id": "8f14e45f-ceea-467e-8e3c-1f1c7c4b6a10",
  "total_cents": 15990
}
```
Headers: `X-Event-Id`, `X-Signature` (HMAC-SHA256), `X-Timestamp`, `X-Webhook-Id`, `Content-Type: application/json` (`[09:44]-[09:45] Diego, Sofia`). Sem `items` do pedido — payload enxuto por design (`[09:43]-[09:44] Diego`).

## Matriz de erros (`WEBHOOK_*`)

| Código | Status HTTP | Cenário |
| --- | --- | --- |
| `WEBHOOK_NOT_FOUND` | 404 | Endpoint de webhook não existe para o `id` informado |
| `WEBHOOK_INVALID_URL` | 400 | URL cadastrada não usa HTTPS (`[09:23] Sofia`) |
| `WEBHOOK_CUSTOMER_NOT_FOUND` | 404 | `customerId` informado não existe |
| `WEBHOOK_SECRET_REQUIRED` | 500 (interno) | Falha ao gerar/persistir secret na criação — condição defensiva |
| `WEBHOOK_INVALID_STATUS_FILTER` | 400 | `statusFilter` vazio ou contém valor fora de `OrderStatus` |
| `WEBHOOK_PAYLOAD_TOO_LARGE` | 422 | Payload do evento excede 64KB no momento da inserção na outbox (`[09:23]-[09:24] Sofia, Diego`) |
| `WEBHOOK_DEAD_LETTER_NOT_FOUND` | 404 | Item de DLQ inexistente para replay |
| `WEBHOOK_ALREADY_INACTIVE` | 409 | Tentativa de desativar um webhook já inativo |

Todos seguem o mesmo formato de resposta de erro definido pelo `errorMiddleware` (`src/middlewares/error.middleware.ts:14-24`): `{ error: { code, message, details? } }`.

## Estratégias de resiliência

- **Timeout**: 10 segundos por chamada HTTP do worker ao endpoint do cliente (`[09:42] Diego`).
- **Retry**: backoff exponencial de 5 tentativas — 1min, 5min, 30min, 2h, 12h (`[09:17] Diego`).
- **Fallback**: esgotadas as tentativas, evento vai para DLQ; não há reenvio automático além desse ponto, apenas replay manual via endpoint admin (`[09:18] Diego`).
- **Circuito de segurança de payload**: eventos cujo payload renderizado excede 64KB são rejeitados na inserção (erro, não truncamento) (`[09:23]-[09:24] Sofia, Diego`).
- **Isolamento de processo**: worker roda fora do processo da API; falha ou reinício de um não afeta o outro (`[09:11] Diego`).

## Observabilidade

- **Métricas**: contagem de eventos inseridos na outbox por status; taxa de sucesso/falha de entrega por endpoint; tempo de resposta (`response_time_ms`, já persistido em `webhook_deliveries`); tamanho da fila de pendentes (`webhook_outbox` com `status = PENDING`); contagem de itens em DLQ.
- **Logs**: via logger Pino já configurado (`src/shared/logger/index.ts`) — log estruturado em cada tentativa de entrega (sucesso/falha, `event_id`, `webhook_endpoint_id`, tempo de resposta), em cada movimentação para DLQ, e em cada replay administrativo (com o `user.id` de quem executou, para auditoria — `[09:36] Sofia`).
- **Tracing**: cada requisição HTTP enviada ao cliente carrega `X-Event-Id`, que serve como identificador de correlação entre o log do worker, a linha em `webhook_deliveries` e o log do lado do cliente (documentado no portal do desenvolvedor — `[09:26] Marcos`).

## Dependências e compatibilidade

- Nenhuma dependência de infraestrutura nova: reaproveita MySQL/Prisma já existentes (`[09:07] Diego`).
- Novo processo Node (`src/worker.ts`) precisa de deploy e monitoramento próprios, análogos aos de `src/server.ts`.
- `PrismaClient` do worker é uma instância separada da API, mesma `DATABASE_URL` (`[09:30] Bruno`).
- Nenhuma mudança de contrato é exigida nos módulos existentes (`orders`, `customers`, `products`, `users`) além do ponto de integração em `changeStatus`.

## Critérios de aceite técnicos

- Uma mudança de status que corresponda ao `statusFilter` de ao menos um webhook ativo do customer resulta em uma linha em `webhook_outbox` na mesma transação da mudança de status.
- Falha na inserção do evento causa rollback completo da transação de `changeStatus` (nenhuma mudança de status parcial).
- Endpoint indisponível gera exatamente 5 tentativas com os intervalos de backoff definidos, e é movido para DLQ após a 5ª falha.
- Toda entrega inclui os headers `X-Event-Id`, `X-Signature`, `X-Timestamp`, `X-Webhook-Id`.
- Rotação de secret mantém a secret anterior válida por exatamente 24h.
- Endpoint de replay de DLQ retorna `403 FORBIDDEN` para roles diferentes de `ADMIN`.

## Riscos e mitigação

| Risco | Mitigação |
| --- | --- |
| Falha na inserção da outbox quebra silenciosamente a garantia de entrega | Inserção dentro da mesma transação Prisma de `changeStatus`; testes de integração cobrindo rollback |
| Worker parado sem alerta gera acúmulo silencioso de eventos pendentes | Métrica de tamanho de fila pendente e alerta operacional (fora do código, mas depende da métrica exposta) |
| Secret vazada compromete integração de um cliente | Secret por endpoint (não global) + rotação com grace period de 24h (ADR-004) |
| Payload volumoso degrada performance de envio | Limite de 64KB com rejeição na inserção (`[09:23]-[09:24]`) |

## Integração com o sistema existente

1. **`src/modules/orders/order.service.ts`** — o método `changeStatus` (linhas 126-179) é estendido para, dentro da transação já existente (`this.prisma.$transaction(async (tx) => {...})`), chamar `publishWebhookEvent(tx, order, from, to)` logo após a atualização de status e antes do commit implícito. Nenhuma outra lógica do método muda; a chamada reaproveita o mesmo `tx` já usado para `tx.order.update` e `tx.orderStatusHistory.create`.
2. **`src/shared/errors/http-errors.ts`** — novas classes de erro do módulo de webhooks (`WebhookNotFoundError`, `WebhookInvalidUrlError`, etc.) estendem `AppError`/as classes existentes (`NotFoundError`, `UnprocessableEntityError`, `ConflictError`), seguindo exatamente o padrão de `InsufficientStockError` e `InvalidStatusTransitionError` já presentes no arquivo, com código prefixado `WEBHOOK_`.
3. **`src/middlewares/auth.middleware.ts`** — o endpoint `POST /admin/webhooks/dead-letter/:id/replay` reaproveita `requireRole('ADMIN')` (linha 49-61) diretamente na definição da rota, sem nenhuma alteração no middleware.
4. **`src/routes/index.ts`** — `buildApiRouter` passa a registrar `router.use('/webhooks', buildWebhookRouter(controllers.webhooks))` e `router.use('/admin/webhooks', buildWebhookAdminRouter(controllers.webhooksAdmin))`, seguindo o mesmo padrão usado para `orders`, `customers`, `products`.
5. **`src/app.ts`** — `buildControllers` passa a instanciar `WebhookRepository`, `WebhookService` e `WebhookController` (e o correspondente para o admin/DLQ), no mesmo padrão de composição usado para `OrderRepository`/`OrderService`/`OrderController` (linhas 42-44).
6. **`package.json`** — novo script `"worker": "tsx watch --env-file=.env src/worker.ts"`, espelhando o script `"dev"` já existente (linha 11), e uma variante de produção equivalente a `"start"` para `dist/worker.js` após o build.
7. **`prisma/schema.prisma`** — quatro novos models (`WebhookEndpoint`, `WebhookOutbox`, `WebhookDeadLetter`, `WebhookDelivery`), seguindo o padrão de `id String @id @default(uuid()) @db.Char(36)` já usado em todos os models existentes (ex: `Order`, `Customer`), com `@@index` equivalentes aos já usados em `Order` (`status`, `createdAt`).
8. **`src/shared/logger/index.ts`** — o worker importa o mesmo logger Pino singleton (`logger`) para registrar tentativas de entrega, sem criar instância ou configuração de logging própria.
