# ADR-001: Padrão Outbox no MySQL para entrega de eventos de webhook

## Status

Aceito

## Contexto

A feature de Webhooks de Notificação de Pedidos precisa disparar uma chamada HTTP para sistemas de clientes (Atlas Comercial, MaxDistribuição, Nova Cargo) sempre que o status de um pedido muda. A mudança de status acontece dentro de `OrderService.changeStatus` (`src/modules/orders/order.service.ts:126-179`), que já executa, em uma única transação Prisma, a atualização da tabela `orders`, a inserção em `order_status_history` e o débito/reposição de `stock_quantity` em `products`.

Bruno (engenheiro do time de Pedidos) apontou que essa transação já é pesada, e que inserir uma chamada HTTP síncrona no meio dela faria qualquer cliente lento ou fora do ar travar a mudança de status de outros pedidos, sem possibilidade de rollback coerente (`[09:04] Bruno`). Diego (engenheiro sênior, Plataforma) propôs o padrão outbox: dentro da mesma transação SQL que já atualiza `orders` e `order_status_history`, inserir uma linha em uma tabela `webhook_outbox` com o evento a ser entregue. Um worker separado lê essa tabela de forma assíncrona e realiza as chamadas HTTP (`[09:06] Diego`).

O time é pequeno e a infraestrutura atual é só MySQL via Prisma — não há Redis, Kafka ou similar em produção.

## Decisão

Adotar o padrão Outbox implementado como uma tabela MySQL (`webhook_outbox`), com inserção feita dentro da mesma transação Prisma (`tx`) que já atualiza o status do pedido em `OrderService.changeStatus`. Se a inserção na outbox falhar, toda a transação sofre rollback — não existe caso possível de status mudar sem o evento correspondente ser registrado (`[09:04] Bruno`, `[09:06] Diego`, `[09:40] Bruno`).

A tabela terá índice em `status` (pendente, processando, falhou, entregue) e em `created_at`, para que o worker leia eficientemente apenas os eventos pendentes mais antigos em lotes pequenos (`[09:07] Diego`). Arquivamento de linhas entregues após 30 dias fica fora do escopo desta feature (`[09:07] Diego`).

## Alternativas Consideradas

- **Fila dedicada (Redis Streams ou similar)**: rejeitada por exigir subir e operar nova infraestrutura (ex: Redis Cluster) para um time pequeno, configurando overengineering para o problema atual (`[09:07] Larissa`, `[09:07] Diego`).
- **Disparo síncrono dentro do `changeStatus`**: rejeitada porque acopla a latência/disponibilidade de sistemas externos à transação crítica de mudança de status, sem estratégia de rollback viável (`[09:04] Bruno`).

## Consequências

**Positivas:**
- Consistência garantida entre mudança de status e emissão do evento, sem necessidade de coordenação distribuída (commit da transação SQL = evento garantido).
- Reaproveita a infraestrutura de banco MySQL/Prisma já existente, sem custo operacional de novos componentes.
- Simples de auditar: a outbox funciona como log de eventos consultável via SQL.

**Negativas:**
- Introduz acoplamento entre o volume de mudanças de status e o crescimento de uma tabela transacional; requer rotina de arquivamento futura (fora de escopo desta fase).
- Sem notificação nativa (MySQL não tem `LISTEN/NOTIFY` como Postgres), a leitura depende de polling do worker, o que implica latência mínima não-zero (ver ADR-002).
