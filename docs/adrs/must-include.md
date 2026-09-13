# Principais Decisões Técnicas — Sistema de Webhooks de Notificação de Pedidos

Resumo das principais decisões técnicas acordadas na reunião técnica registradas a partir de `TRANSCRICAO.md`:

- **Padrão Transacional Outbox no MySQL**: Publicação assíncrona gravando eventos na tabela `webhook_outbox` dentro da mesma transação SQL da alteração de status do pedido (`OrderService.changeStatus`), assegurando consistência atômica sem introduzir nova infraestrutura (ex: Redis).
- **Worker Desacoplado via Polling**: Processamento assíncrono em processo Node.js separado (`src/worker.ts`) consultando a tabela de outbox a cada 2 segundos por polling em batch ordenado por `created_at`, mantendo ordenação por pedido em cenário single-worker e latência abaixo de 10s.
- **Garantia de Entrega At-Least-Once com Desduplicação por Event ID**: Adoção da semântica *at-least-once* com envio do identificador único no cabeçalho `X-Event-Id` (UUID gerado na criação do evento), delegando a responsabilidade de idempotência ao cliente.
- **Política de Retry com Backoff Exponencial e Tabela DLQ Dedicada**: Estratégia de retentativas com até 5 tentativas espaçadas (1m, 5m, 30m, 2h, 12h; janela de ~15h) e encaminhamento de falhas definitivas para a tabela `webhook_dead_letter`, permitindo reprocessamento manual via endpoint administrativo seguro (`POST /admin/webhooks/dead-letter/:id/replay`).
- **Autenticação e Integridade via HMAC-SHA256 com Secret por Endpoint**: Assinatura criptográfica do payload enviada no header `X-Signature` gerada com secret exclusiva por webhook, suportando rotação com janela de carência (*grace period*) de 24 horas para coexistência de chaves.
- **Reaproveitamento Integral dos Padrões da Codebase**: Implementação no padrão modular em `src/modules/webhooks` com Prisma, Schemas Zod, classes de erro herdadas de `AppError`, prefixo de códigos `WEBHOOK_*`, controle de acesso com `requireRole` (role `ADMIN` para replay de DLQ) e logging com `Pino`.
