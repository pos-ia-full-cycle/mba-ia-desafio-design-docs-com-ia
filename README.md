# Sistema de Webhooks de Notificação de Pedidos — Pacote de Design Docs

Este repositório é o fork do desafio [Da Reunião ao Documento: Design Docs Gerados por IA](#enunciado-original), da MBA IA para Devs. O código-fonte do OMS (`src/`, `prisma/`, `tests/`) não foi alterado — a entrega é puramente documental, produzida em `docs/` a partir da transcrição de uma reunião técnica (`TRANSCRICAO.md`) e do código existente.

## Sobre o desafio

O desafio simula uma situação comum em times de engenharia: uma decisão técnica importante foi tomada em uma reunião — Tech Lead, PM, dois engenheiros e uma engenheira de segurança discutindo por ~55 minutos como construir um sistema de webhooks de notificação de pedidos — mas nada disso foi registrado além da transcrição literal da call. A tarefa é transformar essa conversa (mais o código do OMS já existente) em um pacote de documentação técnica acionável: PRD, RFC, FDD, um conjunto de ADRs e um tracker de rastreabilidade.

O ponto central do exercício não é redigir documentos bonitos, mas produzir documentação **rastreável**: cada requisito, decisão ou restrição registrada precisa apontar para uma linha específica da transcrição ou para um arquivo real do código. Nada pode ser inventado — inclusive o que foi discutido e depois descartado (rate limiting adiado, e-mail de alerta fora de escopo, dashboard fora de escopo) precisa aparecer registrado como tal, não como requisito.

## Ferramentas de IA utilizadas

- **Claude Code (Sonnet 5)**, rodando como extensão nativa dentro do VS Code — única ferramenta usada nesta produção. Atuou como agente de ponta a ponta: leu a transcrição e o código-fonte completo do OMS diretamente do disco, e gerou todos os documentos (`docs/PRD.md`, `docs/RFC.md`, `docs/FDD.md`, os 7 ADRs e `docs/TRACKER.md`) em uma sessão contínua, com acesso de leitura/escrita ao repositório em vez de um chat separado com copiar-e-colar.

## Workflow adotado

A produção seguiu a ordem sugerida pelo enunciado original do desafio:

1. **Contextualização**: leitura integral de `TRANSCRICAO.md` e exploração dirigida do código — `order.service.ts` (método `changeStatus` e a transação que ele já executa), `order.status.ts` (máquina de estados), a hierarquia de erros (`app-error.ts`, `http-errors.ts`), os middlewares (`auth.middleware.ts`, `validate.middleware.ts`, `error.middleware.ts`), o logger Pino, `schema.prisma`, e o padrão de módulo usando `customers` como referência (`controller`/`service`/`routes`/`schemas`), além de `app.ts`, `routes/index.ts`, `server.ts` e `package.json` para entender como um novo módulo e um novo processo (`worker`) se encaixariam na composição existente.
2. **ADRs primeiro**: as 6 decisões centrais da reunião (outbox no MySQL, worker em polling, retry+DLQ, HMAC por endpoint, at-least-once com `X-Event-Id`, reuso de padrões) mais uma decisão adicional identificada na conversa pós-reunião (snapshot do payload na inserção) viraram 7 ADRs individuais, cada um com alternativas e trade-offs explícitos extraídos da própria discussão.
3. **RFC**: consolidação da proposta técnica em nível de arquitetura, referenciando os ADRs já escritos, com as alternativas descartadas (síncrono, Redis Streams, retry de 3 tentativas) e as questões deixadas em aberto na reunião (rate limiting, e-mail de alerta, escala do worker).
4. **FDD**: detalhamento de implementação em cima do RFC e dos ADRs — fluxos passo a passo, 7 contratos de endpoint com payloads de exemplo, matriz de erros `WEBHOOK_*`, e a seção obrigatória de integração com 8 arquivos reais do código existente.
5. **PRD**: escrito por último entre os documentos grandes, como consolidação do que já estava decidido em RFC/FDD/ADRs, traduzido para a linguagem de produto (objetivos, métricas, escopo, riscos).
6. **Tracker**: montado ao final, varrendo cada um dos quatro documentos e localizando manualmente, linha a linha da transcrição, o timestamp exato de cada requisito, decisão ou restrição citada.
7. **README**: este documento, escrito por último.

## Prompts customizados

O desafio recomenda "prompts dirigidos, não pedidos genéricos". Dois exemplos que estruturaram a produção nesta sessão:

**1. Diretriz para extração de decisões (usada antes de escrever os ADRs):**
```
Leia a TRANSCRICAO.md por completo. Identifique as decisões técnicas que
foram efetivamente FECHADAS na reunião (não sugestões soltas, não pontos
adiados). Para cada uma, eu preciso saber:
- quem propôs e quem mais participou da discussão
- qual alternativa concorrente foi colocada na mesa e por que foi descartada
- o timestamp [hh:mm] e o nome de quem fala, para cada afirmação relevante

Separe claramente decisões fechadas de: (a) pontos explicitamente descartados,
(b) pontos adiados para fase futura, (c) detalhes técnicos secundários que não
merecem virar ADR isolado. Não misture os três grupos.
```

**2. Diretriz anti-alucinação para o Tracker (usada na etapa final):**
```
Para cada item que você registrou no PRD, RFC, FDD e nos ADRs, encontre a
localização exata de origem: se veio da transcrição, cite [hh:mm] Nome
correspondente à fala literal — não aproxime o timestamp. Se veio do código,
cite o caminho de arquivo real, não um caminho hipotético.

Se para algum item você não conseguir apontar uma localização exata, não
invente uma aproximada: sinalize o item para eu decidir se ele deve ser
reescrito com uma fonte real ou removido do documento.
```

## Iterações e ajustes

O processo não saiu perfeito na primeira geração. Ajustes concretos feitos durante a produção:

1. **Typo introduzido na escrita do ADR-004**: ao redigir a alternativa descartada de rotação sem grace period, um erro de digitação ("seguIndent") passou para o texto. Encontrado e corrigido em uma releitura antes de seguir para o próximo documento.
2. **Citação incorreta no Tracker**: a linha `FDD-ERR-04` (decisão do prefixo `WEBHOOK_` para códigos de erro) inicialmente apontava para `[09:28] Bruno`, mas ao conferir o texto literal da transcrição, o fechamento explícito dessa decisão é de Larissa em `[09:29]` — Bruno apenas usou o prefixo em exemplos um instante antes. Corrigido após checagem linha a linha de cada timestamp citado no tracker.
3. **Risco descartado do Tracker por falta de rastreabilidade**: o PRD listava como risco "worker parado sem detecção gera atraso silencioso na entrega" — uma preocupação de engenharia legítima, mas que nenhum participante da reunião chegou a verbalizar com um timestamp específico. Em vez de forçar uma citação aproximada no tracker só para bater a cota de cobertura, o item foi mantido no PRD (é um risco real de qualquer sistema de worker assíncrono) mas deixado **fora** do tracker, seguindo a orientação do próprio enunciado: "se você não consegue preencher a coluna Localização, é sinal de que aquela informação não tem origem identificável".
4. **Verificação exaustiva de timestamps**: todos os ~85 timestamps citados como fonte `TRANSCRICAO` no tracker foram conferidos individualmente contra o texto literal de `TRANSCRICAO.md` (não apenas contra a memória da leitura inicial), para eliminar timestamps aproximados que são um sintoma comum de alucinação de IA em exercícios como este.

## Como navegar a entrega

Ordem sugerida de leitura (do "porquê" para o "como", do mais alto nível ao mais detalhado):

1. [`docs/PRD.md`](docs/PRD.md) — problema, público, escopo, métricas de sucesso.
2. [`docs/RFC.md`](docs/RFC.md) — proposta técnica em nível de arquitetura, alternativas descartadas, questões em aberto.
3. [`docs/adrs/`](docs/adrs/) — cada decisão arquitetural isolada, com contexto e consequências:
   - [ADR-001](docs/adrs/ADR-001-outbox-pattern-no-mysql.md) — Padrão Outbox no MySQL
   - [ADR-002](docs/adrs/ADR-002-worker-em-processo-separado-com-polling.md) — Worker em processo separado com polling
   - [ADR-003](docs/adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md) — Retry com backoff exponencial e DLQ
   - [ADR-004](docs/adrs/ADR-004-autenticacao-hmac-sha256-com-secret-por-endpoint.md) — HMAC-SHA256 com secret por endpoint
   - [ADR-005](docs/adrs/ADR-005-garantia-at-least-once-com-x-event-id.md) — Garantia at-least-once com X-Event-Id
   - [ADR-006](docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md) — Reuso dos padrões existentes do projeto
   - [ADR-007](docs/adrs/ADR-007-snapshot-do-payload-na-insercao-do-outbox.md) — Snapshot do payload na inserção do outbox
4. [`docs/FDD.md`](docs/FDD.md) — fluxos detalhados, contratos de API, matriz de erros, observabilidade e integração com o código existente.
5. [`docs/TRACKER.md`](docs/TRACKER.md) — rastreabilidade de cada item registrado até a transcrição ou o código.

## Enunciado original

O enunciado completo do desafio (objetivo, requisitos por documento, critérios de aceite) pode ser consultado no repositório base: https://github.com/devfullcycle/mba-ia-desafio-design-docs-com-ia
