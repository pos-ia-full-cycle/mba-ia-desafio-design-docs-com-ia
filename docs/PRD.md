# PRD — Sistema de Webhooks de Notificação de Pedidos

## Resumo e contexto da feature

O OMS hoje não possui nenhum mecanismo de notificação externa: clientes que precisam saber quando o status de um pedido muda são obrigados a consultar `GET /orders` repetidamente. Esta feature adiciona um sistema de webhooks outbound, permitindo que clientes B2B cadastrem endpoints HTTP próprios e recebam automaticamente notificações assíncronas sempre que um pedido deles mudar para um status de interesse.

A decisão de construir a feature foi tomada em reunião técnica entre Tech Lead, PM, engenharia e segurança (`TRANSCRICAO.md`), e este documento consolida o que foi decidido em termos de produto: problema, público, escopo e critérios de sucesso. O detalhamento técnico da solução está no RFC (`docs/RFC.md`) e no FDD (`docs/FDD.md`); as decisões arquiteturais isoladas estão nos ADRs (`docs/adrs/`).

## Problema e motivação

Três clientes B2B — Atlas Comercial, MaxDistribuição e Nova Cargo — solicitaram formalmente notificação em tempo real de mudanças de status de seus pedidos (`[09:00] Marcos`). Hoje eles fazem polling manual em `GET /orders`, o que a equipe de produto descreve como uma integração "lenta e cara" para eles manterem (`[09:00] Marcos`). Esse atrito tem risco de negócio concreto: a Atlas sinalizou que pode migrar para um concorrente caso a entrega não ocorra até o fim do trimestre (`[09:00] Marcos`).

## Público-alvo e cenários de uso

**Público-alvo:** clientes B2B da plataforma que operam sistemas próprios de integração (ex: ERPs, sistemas de logística) e precisam reagir a mudanças de status de pedidos sem depender de polling manual.

**Cenários de uso:**
- Um cliente cadastra um endpoint de webhook para ser notificado quando pedidos dele mudarem para `SHIPPED` e `DELIVERED`, e passa a atualizar automaticamente o status no sistema dele quando recebe a notificação.
- Um cliente rotaciona a secret do webhook periodicamente por política interna de segurança, sem interromper o recebimento de eventos durante a transição.
- Um cliente investiga uma falha de integração consultando o histórico de entregas do seu webhook (sucesso/falha, payload, tempo de resposta).
- Um operador administrativo (`ADMIN`) reprocessa manualmente um evento que esgotou todas as tentativas de entrega e caiu em falha permanente.

## Objetivos e métricas de sucesso

- **Eliminar a necessidade de polling manual**: 100% das mudanças de status assinadas por um cliente (dentro do `statusFilter` cadastrado) devem gerar uma tentativa de entrega via webhook, sem exigir chamadas adicionais do cliente a `GET /orders`.
- **Atender à definição de "tempo real" combinada com os clientes**: latência de entrega (do commit da transação de mudança de status até a primeira tentativa de envio) abaixo de 10 segundos no cenário normal (sem falha do endpoint do cliente) — meta explícita levantada pelo PM junto aos clientes (`[09:02] Marcos`).
- **Reduzir risco de churn**: entregar a feature em produção até o fim do trimestre, prazo comunicado à Atlas Comercial (`[09:00] Marcos`, `[09:45]-[09:47] Marcos, Larissa`), com estimativa interna de 3 sprints incluindo revisão de segurança.
- **Resiliência de entrega**: eventos gerados devem ter cobertura de retry suficiente para sobreviver a indisponibilidades transitórias de até ~15 horas do endpoint do cliente antes de serem considerados falha permanente (`[09:17] Diego`).

## Escopo

### Incluso

- Cadastro, edição, remoção e listagem de webhooks por cliente (CRUD de configuração).
- Filtro de eventos por status de pedido, configurável por webhook.
- Emissão assíncrona de eventos de mudança de status, com garantia de consistência com a transação de negócio.
- Retry automático com backoff exponencial e Dead Letter Queue para falhas persistentes.
- Autenticação de entregas via HMAC-SHA256 com secret exclusiva por endpoint, com suporte a rotação.
- Garantia de entrega at-least-once, com identificador único (`X-Event-Id`) para deduplicação do lado do cliente.
- Histórico de entregas consultável pelo cliente.
- Endpoint administrativo (`ADMIN`) para reprocessamento manual de eventos em falha permanente.

### Fora de escopo

- **Notificação por e-mail em caso de falhas recorrentes de um webhook**: descartada para esta fase; considerada possível fase futura, após medição do impacto real de falhas (`[09:37]-[09:38] Larissa, Marcos`).
- **Rate limiting de envio por cliente**: não será implementado agora; a equipe decidiu observar o comportamento em produção e decidir depois se isso vira um problema real (`[09:38]-[09:39] Diego, Larissa`).
- **Dashboard visual para o cliente gerenciar webhooks**: fora de escopo desta feature — seria um projeto separado do time de frontend; esta entrega é somente de endpoints de API (`[09:39]-[09:40] Marcos, Larissa`).
- **Arquivamento automático de eventos entregues**: rotina de arquivamento de linhas da outbox após 30 dias fica fora do escopo desta feature (`[09:07] Diego`).
- **Escala horizontal do worker com ordering global garantido**: o design assume single-worker; suporte a múltiplos workers com particionamento é problema declarado para o futuro, não para esta entrega (`[09:12]-[09:13] Diego, Bruno`).

## Requisitos funcionais

1. O cliente deve poder cadastrar um webhook informando `customerId`, `url` (HTTPS) e lista de status de interesse; a secret é gerada pela plataforma e devolvida apenas na resposta de criação (`[09:31]-[09:32] Marcos`).
2. O cliente deve poder editar um webhook existente (URL, filtro de status, estado ativo/inativo) (`[09:33] Bruno`).
3. O cliente deve poder remover um webhook cadastrado (`[09:33] Bruno`).
4. O cliente deve poder listar os webhooks cadastrados para um `customerId` (`[09:33] Bruno`).
5. O cliente deve poder rotacionar a secret de um webhook via API, mantendo a secret anterior válida por 24 horas em paralelo (`[09:21] Sofia`).
6. O cliente deve poder consultar o histórico das últimas 100 entregas de um webhook, incluindo sucesso/falha, payload, resposta e tempo de resposta (`[09:34]-[09:35] Marcos`).
7. O sistema deve emitir automaticamente um evento de notificação sempre que um pedido mudar para um status presente no filtro de um webhook ativo do respectivo cliente (`[09:33]-[09:34] Marcos, Bruno`).
8. O sistema deve garantir que a emissão do evento seja consistente com a transação de mudança de status — não deve haver caso em que o status muda e o evento correspondente não é registrado (`[09:40] Bruno`).
9. O sistema deve assinar cada entrega com HMAC-SHA256, permitindo ao cliente validar autenticidade e integridade do payload (`[09:19]-[09:20] Sofia`).
10. O sistema deve reter e reenviar automaticamente eventos não entregues, com backoff exponencial, antes de considerá-los falha permanente (`[09:15]-[09:17] Diego`).
11. O sistema deve identificar cada evento com um identificador único (`event_id`), reenviado em toda tentativa, para que o cliente possa deduplicar entregas repetidas (at-least-once) (`[09:24]-[09:25] Diego`).
12. Um usuário com role `ADMIN` deve poder reprocessar manualmente um evento que esgotou as tentativas automáticas de entrega, com o reprocessamento registrado para auditoria (`[09:18]-[09:19] Diego`, `[09:35]-[09:36] Sofia`).

## Requisitos não funcionais

- **Latência**: entrega inicial (primeira tentativa) em até 10 segundos após a mudança de status, em condições normais de operação (`[09:02] Marcos`).
- **Segurança de transporte**: URLs de webhook cadastradas devem ser HTTPS; cadastro de URL HTTP deve ser recusado na validação (`[09:23] Sofia`).
- **Segurança de payload**: eventos com payload renderizado acima de 64KB devem ser rejeitados, não truncados (`[09:23]-[09:24] Sofia, Diego`).
- **Isolamento de secret**: cada endpoint de webhook deve ter secret própria, nunca compartilhada entre clientes distintos (`[09:21] Sofia`).
- **Confiabilidade**: cobertura de retry deve sobreviver a indisponibilidades de cliente de até ~15 horas antes de mover o evento para falha permanente (`[09:15]-[09:17] Diego`).
- **Sem infraestrutura nova**: a solução deve operar sobre o MySQL/Prisma já existente, sem exigir componentes de infraestrutura adicionais (ex: filas dedicadas) (`[09:07] Diego`, `[09:07] Larissa`).
- **Timeout de rede**: chamadas HTTP a endpoints de cliente devem expirar em 10 segundos (`[09:42] Diego`).

## Decisões e trade-offs principais

As decisões arquiteturais centrais estão registradas individualmente nos ADRs correspondentes:

- Emissão via padrão outbox no MySQL, trocando simplicidade operacional por uma dependência de polling em vez de notificação nativa ([ADR-001](adrs/ADR-001-outbox-pattern-no-mysql.md)).
- Worker em processo separado com polling de 2s, trocando reatividade instantânea por isolamento operacional e latência previsível ([ADR-002](adrs/ADR-002-worker-em-processo-separado-com-polling.md)).
- Retry de 5 tentativas com backoff até 12h, trocando "matar rápido" por resiliência a indisponibilidades reais já observadas em clientes ([ADR-003](adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md)).
- Garantia at-least-once (não exactly-once), trocando simplicidade de implementação por responsabilidade de deduplicação transferida ao cliente ([ADR-005](adrs/ADR-005-garantia-at-least-once-com-x-event-id.md)).

## Dependências

- Módulo de pedidos existente (`src/modules/orders/`), especificamente o método `changeStatus`, ponto único de origem dos eventos.
- Infraestrutura de banco MySQL/Prisma já provisionada — nenhuma dependência de infraestrutura nova.
- Disponibilidade da Sofia (segurança) para revisão dedicada de HMAC e geração de secret antes do deploy — pelo menos dois dias úteis reservados (`[09:46] Sofia`).
- Portal de desenvolvedor (responsabilidade do Marcos) para documentar aos clientes o comportamento de at-least-once/deduplicação e o processo de integração (`[09:26] Marcos`, `[09:40] Marcos`).

## Riscos e mitigação

| Risco | Probabilidade | Impacto | Mitigação |
| --- | --- | --- | --- |
| Cliente Atlas Comercial migra para concorrente por atraso na entrega | Média | Alto — perda de conta B2B relevante | Escopo restrito ao essencial acordado em reunião; estimativa de 3 sprints comunicada e prazo confirmado com o cliente (`[09:45]-[09:47] Marcos, Larissa`) |
| Vazamento de secret compromete a integração de um cliente | Média (já ocorreu antes com outro cliente) | Alto — dados de pedidos expostos a terceiros não autorizados | Secret exclusiva por endpoint (não global) + rotação com grace period de 24h ([ADR-004](adrs/ADR-004-autenticacao-hmac-sha256-com-secret-por-endpoint.md)) |
| Worker parado sem detecção atrasa silenciosamente todas as entregas | Média | Médio — clientes deixam de receber notificações sem sinal de erro imediato | Métricas de tamanho da fila pendente e logs estruturados (ver FDD, seção Observabilidade) |
| Payload de evento excessivamente grande degrada envio ou processamento | Baixa | Baixo | Limite de 64KB com rejeição na inserção, não truncamento (`[09:23]-[09:24] Sofia, Diego`) |

## Critérios de aceitação

- Cliente autenticado consegue cadastrar, editar, remover e listar webhooks via API, recebendo a secret apenas na criação.
- Mudança de status de um pedido gera evento de webhook se e somente se algum webhook ativo do cliente estiver assinando aquele status.
- Nenhuma mudança de status ocorre sem o evento correspondente ser registrado (garantia transacional).
- Falha de entrega aciona retry com os intervalos de backoff definidos, e falha após 5 tentativas resulta em item na DLQ.
- Toda entrega ao cliente inclui assinatura HMAC-SHA256 válida e identificador único de evento.
- Endpoint de replay de itens da DLQ só é acessível a usuários com role `ADMIN`.
- Rotação de secret mantém a secret anterior válida por exatamente 24 horas após a rotação.

## Estratégia de testes e validação

- **Testes unitários**: lógica de filtragem de eventos por `statusFilter`, cálculo de backoff exponencial, geração e validação de assinatura HMAC-SHA256.
- **Testes de integração**: garantir que uma falha na inserção do evento na outbox causa rollback completo da transação de `changeStatus` (seguindo o padrão de testes já presente em `tests/orders.test.ts`); ciclo completo de retry até DLQ simulando endpoint indisponível; validação de contrato dos endpoints via `supertest`, no mesmo padrão de `tests/auth.test.ts`.
- **Testes de contrato**: verificação de payloads de exemplo, headers obrigatórios (`X-Event-Id`, `X-Signature`, `X-Timestamp`, `X-Webhook-Id`) e códigos de status HTTP descritos no FDD.
- **Revisão de segurança dedicada**: revisão manual da Sofia sobre a implementação de HMAC e geração/rotação de secret antes do deploy em produção, com pelo menos dois dias úteis reservados (`[09:46] Sofia`).
