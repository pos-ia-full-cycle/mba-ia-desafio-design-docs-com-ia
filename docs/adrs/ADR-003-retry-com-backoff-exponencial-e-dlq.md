# ADR-003: Retry com backoff exponencial (5 tentativas) e Dead Letter Queue em tabela separada

## Status

Aceito

## Contexto

Endpoints de clientes podem estar temporariamente indisponíveis (manutenção, instabilidade). O time discutiu quantas tentativas de reenvio fazer, com que espaçamento, e o que fazer após esgotar as tentativas.

Diego propôs backoff exponencial com teto de tentativas, movendo para uma "falha permanente" (DLQ) depois disso (`[09:15] Diego`). Bruno defendeu inicialmente 3 tentativas por ser "mais agressivo" (`[09:16] Bruno`), mas Diego citou um caso real de cliente com indisponibilidade de duas horas em manutenção planejada, argumentando que 3 tentativas matariam o evento cedo demais (`[09:16] Diego`). A equipe fechou em 5 tentativas.

Para o destino final de eventos que esgotam as tentativas, Diego propôs uma tabela separada (`webhook_dead_letter`) em vez de apenas marcar como "failed" na própria outbox, para manter a outbox principal limpa e ter uma tabela dedicada de evidência para debug e reprocessamento (`[09:18] Diego`).

## Decisão

Implementar retry com backoff exponencial de 5 tentativas, com intervalos de 1 minuto, 5 minutos, 30 minutos, 2 horas e 12 horas entre tentativas — cobrindo uma janela de aproximadamente 15 horas entre a primeira falha e a última tentativa (`[09:17] Diego`, `[09:17] Marcos`). Esgotadas as 5 tentativas, o evento é movido para a tabela `webhook_dead_letter`, contendo o payload, o motivo da falha e o timestamp (`[09:18] Diego`).

Reprocessamento de itens da DLQ é manual, via endpoint administrativo `POST /admin/webhooks/dead-letter/:id/replay`, que recoloca o evento na outbox como pendente (`[09:18] Diego`, `[09:35] Diego`). Esse endpoint exige role `ADMIN` (ver ADR referente a reuso de padrões existentes) e deve registrar log de auditoria de quem executou o replay (`[09:35]–[09:36] Sofia, Larissa`).

## Alternativas Consideradas

- **3 tentativas com backoff mais agressivo**: descartada porque uma janela curta (~30 minutos) mataria eventos durante indisponibilidades legítimas e já observadas de clientes reais (`[09:16] Diego`).
- **Retry indefinido**: descartado por deixar eventos pendurados indefinidamente caso o cliente tenha desistido da integração ou o endpoint nunca mais volte, sem sinalização clara de falha permanente (`[09:15] Diego`).
- **Marcar falha permanente como status na própria tabela outbox** (sem tabela separada): descartada por poluir a leitura/índices da outbox principal, que deve conter majoritariamente eventos ativos (`[09:18] Diego`).

## Consequências

**Positivas:**
- Cobre indisponibilidades realistas de clientes (até ~15h) sem exigir intervenção manual.
- DLQ separada mantém a outbox operacional limpa e fornece evidência estruturada para diagnóstico.
- Reprocessamento manual e auditável evita reenvio automático de eventos potencialmente problemáticos sem revisão humana.

**Negativas:**
- Eventos que falham definitivamente exigem ação manual via endpoint admin — não há reprocessamento automático além da DLQ.
- Backoff de 12h na última tentativa significa que, no pior caso, um cliente só recebe a notificação quase 15h depois da mudança de status (ou nunca, se cair em DLQ) — aceitável para o requisito atual, mas é um trade-off explícito de latência versus robustez.
