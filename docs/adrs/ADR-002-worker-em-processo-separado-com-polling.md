# ADR-002: Worker de entrega em processo separado, com polling de 2 segundos

## Status

Aceito

## Contexto

Os eventos gravados na `webhook_outbox` (ADR-001) precisam ser lidos e entregues aos endpoints dos clientes. A equipe discutiu duas questões: (1) onde esse processamento roda, e (2) como ele descobre novos eventos.

Diego destacou que o worker precisa rodar como processo separado da API, para que um restart da API não derrube o worker (`[09:11] Diego`). Bruno perguntou se seria possível usar triggers de banco para reatividade; Diego esclareceu que MySQL não tem um mecanismo equivalente ao `LISTEN/NOTIFY` do Postgres — um trigger só executa SQL, não notifica um processo externo, e improvisar isso (escrever em arquivo, chamar um endpoint) seria artificial (`[09:09] Bruno`, `[09:09] Diego`). O requisito de negócio vindo do Marcos é que os clientes B2B consideram "tempo real" qualquer coisa abaixo de 10 segundos (`[09:02] Marcos`).

## Decisão

Implementar o worker como um novo entry-point do processo Node, análogo a `src/server.ts` (`[09:11] Larissa`), adicionando `src/worker.ts` e um script `npm run worker` no `package.json`. O worker roda em loop de polling, verificando a cada 2 segundos os eventos pendentes mais antigos na outbox, processando-os e marcando o resultado (`[09:09] Diego`, `[09:10] Marcos`). Ele se conecta ao mesmo banco (mesma `DATABASE_URL`) mas instancia seu próprio `PrismaClient`, por ser um processo Node distinto (`[09:11] Bruno`, `[09:30] Bruno`).

A ordenação de entrega é garantida apenas por `order_id` e apenas enquanto houver um único worker ativo (processamento em ordem de `created_at` da outbox); não há garantia de ordering global entre pedidos diferentes. Essa é uma limitação conhecida e documentada, não um requisito violado — os clientes nunca pediram ordering global, apenas consistência por pedido (`[09:12]–[09:14] Diego, Bruno, Marcos`).

## Alternativas Consideradas

- **Worker dentro do mesmo processo da API** (ex: `setInterval` no `server.ts`): rejeitada porque um restart/deploy da API mataria o worker junto, e picos de carga da API afetariam a entrega de webhooks (`[09:11] Diego`).
- **Reatividade via trigger de banco (event-driven)**: rejeitada porque MySQL não oferece um mecanismo nativo de notificação de processos externos; a alternativa de improvisar (arquivo, endpoint) foi descartada por complexidade desnecessária frente ao requisito de latência (`[09:09] Diego`).
- **Múltiplos workers em paralelo desde o início**: descartada por enquanto — perderia a ordenação implícita por `order_id`. Fica registrado como problema a resolver no futuro (particionamento por `order_id` ou lock pessimista), não neste momento (`[09:13] Diego`).

## Consequências

**Positivas:**
- Isolamento de falhas: reinício ou crash da API não afeta o worker, e vice-versa.
- Latência máxima previsível (2s de polling), bem dentro do requisito de "abaixo de 10 segundos" definido pelos clientes.
- Escala e deploy independentes do worker em relação à API.

**Negativas:**
- Não há garantia de ordering global entre pedidos distintos quando múltiplos workers rodarem no futuro — limitação documentada, não resolvida nesta fase.
- Adiciona um processo a mais para operar, monitorar e fazer deploy (novo script, novo processo em produção).
- Latência mínima de até 2 segundos é inerente ao modelo de polling (não é "tempo real" no sentido estrito, mas atende ao requisito acordado).
