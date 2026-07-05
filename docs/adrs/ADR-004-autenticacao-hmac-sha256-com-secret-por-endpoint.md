# ADR-004: Autenticação de webhooks via HMAC-SHA256 com secret por endpoint e rotação com grace period

## Status

Aceito

## Contexto

Os webhooks expõem dados de pedidos para sistemas fora da infraestrutura da empresa. Sofia (engenharia de segurança) levantou a necessidade de o cliente conseguir validar que a requisição realmente veio da plataforma e que o payload não foi adulterado em trânsito (`[09:19] Sofia`).

O padrão de mercado proposto foi assinar o corpo da requisição com uma secret compartilhada, usando HMAC-SHA256, e enviar a assinatura em um header (`X-Signature`) para o cliente validar do lado dele (`[09:20] Sofia`). Sofia também exigiu que cada endpoint de webhook tenha uma secret única — não uma secret global da plataforma — para que o vazamento de uma secret não comprometa todos os clientes (`[09:21] Sofia`). Diego citou um incidente real em que um cliente vazou a secret em log de aplicação, reforçando a necessidade de rotação (`[09:22] Diego`).

## Decisão

Cada endpoint de webhook cadastrado (linha na tabela de configuração, com `url`, `secret`, `customer_id` e estado ativo) recebe sua própria secret, gerada pela plataforma e devolvida na criação (`[09:21] Sofia`, `[09:31]-[09:32] Marcos`). A assinatura do payload usa HMAC-SHA256 e é enviada no header `X-Signature`.

A secret é rotacionável via endpoint da API. Durante a rotação, a secret antiga permanece válida em paralelo por 24 horas (grace period), dando tempo para o cliente migrar seus sistemas antes de a secret antiga ser invalidada (`[09:21] Sofia`).

TLS é obrigatório: URLs de webhook cadastradas com `http://` são recusadas na validação do schema (Zod), não sendo tratada como uma decisão arquitetural separada, apenas uma regra de validação (`[09:23] Sofia`).

## Alternativas Consideradas

- **Secret global única para toda a plataforma**: rejeitada porque um vazamento comprometeria a autenticidade de webhooks para todos os clientes simultaneamente (`[09:21] Sofia`).
- **Rotação sem grace period (secret antiga invalidada imediatamente)**: implicitamente descartada — a equipe já tinha experiência de um cliente vazando secret e precisando de uma janela segura para migrar, o que motivou o grace period de 24h (`[09:22] Diego`).
- **Outro algoritmo de assinatura (ex: RSA/assinatura assimétrica)**: não chegou a ser proposto como alternativa séria; HMAC-SHA256 foi adotado diretamente por ser o padrão de mercado, mais simples de implementar em ambos os lados (`[09:20] Sofia`).

## Consequências

**Positivas:**
- Isolamento de blast radius: vazamento de uma secret afeta um único endpoint/cliente, não a plataforma inteira.
- Rotação com grace period permite operação segura sem downtime de integração para o cliente.
- HMAC-SHA256 é amplamente suportado por bibliotecas cliente, reduzindo fricção de adoção.

**Negativas:**
- Aumenta a superfície de gerenciamento: cada endpoint precisa armazenar e proteger sua própria secret (armazenamento seguro é responsabilidade adicional da plataforma).
- Durante o grace period de 24h, duas secrets são simultaneamente válidas para o mesmo endpoint, o que exige lógica de validação que tente ambas até a expiração da antiga.
