# Tracker de Rastreabilidade

Este documento fornece a matriz de rastreabilidade completa conectando cada requisito, decisão arquitetural, restrição técnica e contrato público documentados no pacote de design docs ([docs/PRD.md](file:///home/raphael/workspace/pos-ia/desafios/mba-ia-desafio-design-docs-com-ia/docs/PRD.md), [docs/RFC.md](file:///home/raphael/workspace/pos-ia/desafios/mba-ia-desafio-design-docs-com-ia/docs/RFC.md), [docs/FDD.md](file:///home/raphael/workspace/pos-ia/desafios/mba-ia-desafio-design-docs-com-ia/docs/FDD.md) e [docs/adrs/](file:///home/raphael/workspace/pos-ia/desafios/mba-ia-desafio-design-docs-com-ia/docs/adrs)) à sua respectiva fonte de origem na transcrição da reunião técnica ([TRANSCRICAO.md](file:///home/raphael/workspace/pos-ia/desafios/mba-ia-desafio-design-docs-com-ia/TRANSCRICAO.md)) ou no código-fonte da aplicação base (`src/` e `prisma/`).

---

## Matriz de Rastreabilidade

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **PRD-MOT-01** | `docs/PRD.md` | Motivação | Demanda mandatória de clientes Atlas, MaxDistribuição e Nova Cargo por notificações em tempo real | `TRANSCRICAO` | `[09:00] Marcos` |
| **PRD-MOT-02** | `docs/PRD.md` | Motivação | Clientes fazem polling constante em GET /orders gerando integração lenta e cara | `TRANSCRICAO` | `[09:00] Marcos` |
| **PRD-MOT-03** | `docs/PRD.md` | Restrição de Negócio | Risco de perda de contrato (churn) com a Atlas Comercial caso não entregue até o fim do trimestre | `TRANSCRICAO` | `[09:00] Marcos` |
| **PRD-OBJ-01** | `docs/PRD.md` | Requisito Não Funcional | Notificação em tempo real definida com latência máxima ponta a ponta abaixo de 10 segundos | `TRANSCRICAO` | `[09:02] Marcos` |
| **PRD-ESC-01** | `docs/PRD.md` | Restrição | Escopo exclusivo para notificações de saída (outbound webhooks) | `TRANSCRICAO` | `[09:02] Marcos` |
| **PRD-ESC-02** | `docs/PRD.md` | Restrição | Notificações de entrada (inbound webhooks) deliberadamente fora de escopo | `TRANSCRICAO` | `[09:03] Sofia` |
| **PRD-FR-01** | `docs/PRD.md` | Requisito Funcional | Cadastro de webhook via endpoint POST com url, secret gerada pelo sistema, lista de status e customerId | `TRANSCRICAO` | `[09:31] Marcos` |
| **PRD-FR-02** | `docs/PRD.md` | Requisito Funcional | Listagem de webhooks configurados por customerId via GET /webhooks | `TRANSCRICAO` | `[09:33] Bruno` |
| **PRD-FR-03** | `docs/PRD.md` | Requisito Funcional | Edição parcial de configuração de webhook (url, eventos, ativo) via PATCH /webhooks/:id | `TRANSCRICAO` | `[09:33] Bruno` |
| **PRD-FR-04** | `docs/PRD.md` | Requisito Funcional | Exclusão de webhook cadastrado via DELETE /webhooks/:id | `TRANSCRICAO` | `[09:33] Bruno` |
| **PRD-FR-05** | `docs/PRD.md` | Requisito Funcional | Filtro de eventos selecionando quais status de pedidos o webhook deseja receber | `TRANSCRICAO` | `[09:33] Marcos` |
| **PRD-FR-06** | `docs/PRD.md` | Requisito Funcional | Filtragem na inserção da outbox para não persistir eventos sem inscritos interessados | `TRANSCRICAO` | `[09:34] Bruno` |
| **PRD-FR-07** | `docs/PRD.md` | Requisito Funcional | Histórico de entregas dos últimos 100 webhooks enviados via GET /webhooks/:id/deliveries | `TRANSCRICAO` | `[09:34] Marcos` |
| **PRD-FR-08** | `docs/PRD.md` | Requisito Funcional | Endpoint de replay manual de mensagens da DLQ POST /admin/webhooks/dead-letter/:id/replay | `TRANSCRICAO` | `[09:35] Diego` |
| **PRD-FR-09** | `docs/PRD.md` | Requisito Funcional | Rotação de secret com endpoint dedicado mantendo chave antiga válida por 24 horas | `TRANSCRICAO` | `[09:21] Sofia` |
| **PRD-FR-10** | `docs/PRD.md` | Requisito Funcional | Despacho de payload enxuto com dados da order e sem array de items | `TRANSCRICAO` | `[09:43] Diego` |
| **PRD-FR-11** | `docs/PRD.md` | Requisito Funcional | Injeção dos cabeçalhos X-Event-Id, X-Signature, X-Timestamp, X-Webhook-Id e Content-Type | `TRANSCRICAO` | `[09:44] Diego` |
| **PRD-OOS-01** | `docs/PRD.md` | Fora de Escopo | Notificação de falha para o cliente via e-mail descartada para esta fase | `TRANSCRICAO` | `[09:37] Larissa` |
| **PRD-OOS-02** | `docs/PRD.md` | Fora de Escopo | Rate limiting de envio para clientes deixado fora de escopo para observação em produção | `TRANSCRICAO` | `[09:39] Larissa` |
| **PRD-OOS-03** | `docs/PRD.md` | Fora de Escopo | Painel/dashboard visual para gerenciamento de webhooks delegado ao time de frontend | `TRANSCRICAO` | `[09:40] Larissa` |
| **PRD-NFR-01** | `docs/PRD.md` | Requisito Não Funcional | Obrigatoriedade estrita de transporte seguro HTTPS na URL do webhook | `TRANSCRICAO` | `[09:23] Sofia` |
| **PRD-NFR-02** | `docs/PRD.md` | Requisito Não Funcional | Limite máximo de tamanho de payload fixado em 64KB com erro se ultrapassar | `TRANSCRICAO` | `[09:24] Diego` |
| **PRD-NFR-03** | `docs/PRD.md` | Requisito Não Funcional | Timeout de chamada HTTP externa do worker fixado em 10 segundos | `TRANSCRICAO` | `[09:42] Diego` |
| **PRD-PRAZO-01** | `docs/PRD.md` | Restrição de Negócio | Prazo total de desenvolvimento estimado em três sprints incluindo revisão de segurança | `TRANSCRICAO` | `[09:47] Larissa` |
| **PRD-SEG-01** | `docs/PRD.md` | Restrição | Janela obrigatória de pelo menos dois dias úteis para revisão de código pela equipe de segurança | `TRANSCRICAO` | `[09:46] Sofia` |
| **RFC-ALT-01** | `docs/RFC.md` | Alternativa Descartada | Disparo síncrono rejeitado por travar transações de pedidos e causar efeito dominó | `TRANSCRICAO` | `[09:04] Bruno` |
| **RFC-ALT-02** | `docs/RFC.md` | Alternativa Descartada | Mensageria dedicada (Redis Streams / Kafka) rejeitada por sobre-engenharia para equipe pequena | `TRANSCRICAO` | `[09:07] Diego` |
| **RFC-OPEN-01** | `docs/RFC.md` | Questão em Aberto | Avaliação futura de controle de vazão de saída (rate limiting) após monitoramento em produção | `TRANSCRICAO` | `[09:39] Diego` |
| **RFC-OPEN-02** | `docs/RFC.md` | Questão em Aberto | Alertas proativos por e-mail em caso de falha recorrente postergados para próxima fase | `TRANSCRICAO` | `[09:37] Marcos` |
| **RFC-OPEN-03** | `docs/RFC.md` | Questão em Aberto | Escalabilidade horizontal com múltiplos workers mantendo ordenação via lock ou particionamento | `TRANSCRICAO` | `[09:13] Diego` |
| **RFC-OPEN-04** | `docs/RFC.md` | Questão em Aberto | Endurecimento futuro de permissões no CRUD de webhooks | `TRANSCRICAO` | `[09:37] Sofia` |
| **ADR-001** | `docs/adrs/ADR-001-padrao-transacional-outbox-no-mysql.md` | Decisão | Padrão Transacional Outbox gravando na tabela webhook_outbox na mesma transação SQL | `TRANSCRICAO` | `[09:08] Larissa` |
| **ADR-001-SNAP**| `docs/adrs/ADR-001-padrao-transacional-outbox-no-mysql.md` | Decisão | Evento da outbox gravado como snapshot JSON renderizado no momento da inserção | `TRANSCRICAO` | `[09:52] Larissa` |
| **ADR-001-UUID**| `docs/adrs/ADR-001-padrao-transacional-outbox-no-mysql.md` | Decisão | Identificador da tabela outbox padronizado como UUID v4 seguindo o projeto | `TRANSCRICAO` | `[09:51] Larissa` |
| **ADR-001-RET** | `docs/adrs/ADR-001-padrao-transacional-outbox-no-mysql.md` | Decisão | Retenção e expurgo planejado de eventos entregues após 30 dias | `TRANSCRICAO` | `[09:08] Diego` |
| **ADR-002** | `docs/adrs/ADR-002-worker-desacoplado-via-polling.md` | Decisão | Worker assíncrono executando polling a cada 2 segundos lendo lotes da outbox | `TRANSCRICAO` | `[09:10] Larissa` |
| **ADR-002-PROC**| `docs/adrs/ADR-002-worker-desacoplado-via-polling.md` | Decisão | Worker rodando como processo separado do sistema operacional (src/worker.ts) | `TRANSCRICAO` | `[09:11] Diego` |
| **ADR-002-ORD** | `docs/adrs/ADR-002-worker-desacoplado-via-polling.md` | Decisão | Ordenação sequencial mantida por created_at e order_id enquanto for single-worker | `TRANSCRICAO` | `[09:13] Larissa` |
| **ADR-003** | `docs/adrs/ADR-003-garantia-entrega-at-least-once-com-desduplicacao-event-id.md` | Decisão | Garantia de entrega at-least-once com cabeçalho X-Event-Id (UUID) para desduplicação no receptor | `TRANSCRICAO` | `[09:26] Larissa` |
| **ADR-004** | `docs/adrs/ADR-004-politica-retry-backoff-exponencial-tabela-dlq.md` | Decisão | Política de 5 retries com backoff exponencial (1m, 5m, 30m, 2h, 12h) totalizando ~15h | `TRANSCRICAO` | `[09:17] Larissa` |
| **ADR-004-DLQ** | `docs/adrs/ADR-004-politica-retry-backoff-exponencial-tabela-dlq.md` | Decisão | Tabela dedicada webhook_dead_letter para isolamento de eventos esgotados | `TRANSCRICAO` | `[09:18] Diego` |
| **ADR-004-REP** | `docs/adrs/ADR-004-politica-retry-backoff-exponencial-tabela-dlq.md` | Decisão | Rota de replay de mensagens da DLQ exigindo obrigatoriamente a role ADMIN | `TRANSCRICAO` | `[09:36] Sofia` |
| **ADR-005** | `docs/adrs/ADR-005-autenticacao-integridade-hmac-sha256-secret-por-endpoint.md` | Decisão | Autenticação e integridade via HMAC-SHA256 com secret única por endpoint de cliente | `TRANSCRICAO` | `[09:22] Sofia` |
| **ADR-005-ROT** | `docs/adrs/ADR-005-autenticacao-integridade-hmac-sha256-secret-por-endpoint.md` | Decisão | Suporte a rotação de secret com período de carência de 24 horas | `TRANSCRICAO` | `[09:22] Sofia` |
| **ADR-006** | `docs/adrs/ADR-006-reaproveitamento-padroes-codebase.md` | Decisão | Reaproveitamento integral dos padrões da codebase (AppError, Pino, schemas Zod e módulos) | `TRANSCRICAO` | `[09:30] Larissa` |
| **ADR-006-MOD** | `docs/adrs/ADR-006-reaproveitamento-padroes-codebase.md` | Decisão | Estrutura modular da nova funcionalidade concentrada em src/modules/webhooks | `TRANSCRICAO` | `[09:28] Bruno` |
| **ADR-006-ERR** | `docs/adrs/ADR-006-reaproveitamento-padroes-codebase.md` | Decisão | Padronização dos códigos de erro com prefixo WEBHOOK_ | `TRANSCRICAO` | `[09:29] Larissa` |
| **FDD-INT-01** | `docs/FDD.md` | Integração de Código | Inserção na outbox acoplada à transação de changeStatus no serviço de pedidos | `CODIGO` | `src/modules/orders/order.service.ts` |
| **FDD-INT-02** | `docs/FDD.md` | Integração de Código | Função pura publishWebhookEvent recebendo o cliente transacional tx | `TRANSCRICAO` | `[09:41] Bruno` |
| **FDD-INT-03** | `docs/FDD.md` | Integração de Código | Instanciação de cliente PrismaClient dedicado por processo para isolar conexões | `CODIGO` | `src/config/database.ts` |
| **FDD-INT-04** | `docs/FDD.md` | Integração de Código | Guarda de rota com validação de perfil requireRole('ADMIN') no endpoint de DLQ replay | `CODIGO` | `src/middlewares/auth.middleware.ts` |
| **FDD-INT-05** | `docs/FDD.md` | Integração de Código | Criação de erros operacionais WEBHOOK_* herdando da classe base AppError | `CODIGO` | `src/shared/errors/app-error.ts` |
| **FDD-INT-06** | `docs/FDD.md` | Integração de Código | Instrumentação de métricas e diagnóstico estruturado via utilitário singleton de logger Pino | `CODIGO` | `src/shared/logger/index.ts` |
| **FDD-INT-07** | `docs/FDD.md` | Integração de Código | Interceptação global de exceções pelo errorMiddleware central sem necessidade de alteração | `CODIGO` | `src/middlewares/error.middleware.ts` |
| **FDD-MOD-01** | `docs/FDD.md` | Integração de Código | Modelos relacionais de pedidos, histórico e clientes integrados ao schema Prisma | `CODIGO` | `prisma/schema.prisma` |
