# Potential ADR: Padrão Transacional Outbox no MySQL

**Module**: WEBHOOKS
**Category**: Architecture
**Priority**: Must Document (Score: 145 / Override: docs/adrs/must-include.md)
**Date Identified**: 2026-09-13

---

## Existing ADR Context

ℹ️ **RELATED DECISIONS**

This decision relates to:
- **Banco de Dados Relacional MySQL 8.0** (`INFRA`): Provedor de banco de dados onde a tabela de outbox residirá.
- **ORM Prisma** (`INFRA`): Mecanismo de execução de transações interativas (`$transaction`) onde o evento será persistido.
- **Worker Desacoplado via Polling** (`WEBHOOKS`): Processo consumidor que lerá as mensagens gravadas pela outbox.

**When creating formal ADR**: Reference these in Related ADRs section.

---

## What Was Identified

Identificou-se a decisão de adotar o padrão arquitetural *Transactional Outbox* para a publicação assíncrona de eventos de alteração de status de pedidos no MySQL. Em vez de disparar requisições HTTP síncronas para os clientes no momento em que o status do pedido é alterado, o sistema gravará um registro com o evento na tabela `webhook_outbox`.

Essa gravação ocorrerá atomicamente dentro da mesma transação SQL (`$transaction`) já existente no método [`OrderService.changeStatus`](file:///home/raphael/workspace/pos-ia/desafios/mba-ia-desafio-design-docs-com-ia/src/modules/orders/order.service.ts#L126-L179), que atualiza o pedido em `orders`, grava o histórico em `order_status_history` e realiza o débito/estorno de estoque em `products`. Caso a transação do pedido falhe ou sofra rollback, o evento não será persistido; se a transação for commitada com sucesso, a publicação do evento estará garantida de forma durável.

A decisão foi formalizada na reunião técnica registrada em `TRANSCRICAO.md` e ratificada em `docs/adrs/must-include.md`, enfatizando que a infraestrutura existente do MySQL 8.0 deve ser reaproveitada integralmente sem a necessidade de introduzir brokers adicionais (como Redis Streams, RabbitMQ ou Kafka).

## Why This Might Deserve an ADR

- **Impact**: Garante consistência dual (*dual-write problem*) e confiabilidade de ponta a ponta entre a persistência do estado do pedido no banco de dados e a emissão de notificações assíncronas para clientes B2B.
- **Trade-offs**: Evita complexidade operacional de subir e gerenciar novos clusters de mensageria (ex: Redis ou Kafka), mas introduz carga adicional de I/O de escrita e leitura na instância existente do MySQL e exige processo assíncrono para consumo e expurgo.
- **Complexity**: Média. Requer criação da tabela `webhook_outbox` com índices adequados em status e `createdAt`, além da injeção do cliente transacional (`tx: Prisma.TransactionClient`) para enfileirar o evento.
- **Team Knowledge**: Crítico. Toda a equipe de desenvolvimento precisa entender que disparos de eventos não devem ser feitos via chamadas HTTP síncronas nos serviços de domínio, mas sim através da tabela outbox.
- **Future Implications**: Estabelece o padrão de publicação de eventos da plataforma, podendo ser estendido no futuro para outros domínios sem ruptura arquitetural.

## Evidence Found in Codebase

### Key Files
- [`src/modules/orders/order.service.ts`](file:///home/raphael/workspace/pos-ia/desafios/mba-ia-desafio-design-docs-com-ia/src/modules/orders/order.service.ts#L126-L179) - Linhas 126-179
  - Mostra a transação interativa do Prisma (`this.prisma.$transaction(async (tx) => { ... })`) onde a persistência do evento na outbox será inserida.
- [`prisma/schema.prisma`](file:///home/raphael/workspace/pos-ia/desafios/mba-ia-desafio-design-docs-com-ia/prisma/schema.prisma#L74-L131) - Linhas 74-131
  - Modelos `Order` e `OrderStatusHistory` que participam da transação de alteração de status.
- [`TRANSCRICAO.md`](file:///home/raphael/workspace/pos-ia/desafios/mba-ia-desafio-design-docs-com-ia/TRANSCRICAO.md#L44-L58) - Linhas 44-58 e 238-245
  - Discussão entre Diego, Larissa e Bruno definindo a obrigatoriedade da transação atômica e descartando Redis por overengineering.
- [`docs/adrs/must-include.md`](file:///home/raphael/workspace/pos-ia/desafios/mba-ia-desafio-design-docs-com-ia/docs/adrs/must-include.md#L5) - Linha 5
  - Requisito mandatório de documentação técnica.

### Code Evidence
```typescript
// Exemplo em src/modules/orders/order.service.ts:131-168
async changeStatus(
  id: string,
  input: UpdateOrderStatusInput,
  userId: string,
): Promise<OrderWithRelations> {
  return this.prisma.$transaction(async (tx) => {
    // ... validações de transição e estoque ...
    await tx.order.update({ where: { id }, data: { status: to } });
    await tx.orderStatusHistory.create({
      data: {
        orderId: id,
        fromStatus: from,
        toStatus: to,
        changedById: userId,
        reason: input.reason ?? null,
      },
    });

    // Ponto de integração do Transactional Outbox:
    // await publishWebhookEvent(tx, order, from, to);
    // ...
```

### Impact Analysis
- Introduced: Introduzido no design em 2026-09-13 baseado na arquitetura transacional commitada em 2026-06-24 (`7ef4317`).
- Modified: Design acordado após análise da transação do `OrderService`.
- Last change: 2026-09-13 (especificação técnica da feature de webhooks).
- Affects: Módulos `ORDERS`, `WEBHOOKS` e schema do banco `prisma/schema.prisma`.
- Recent themes: "transação atômica", "consistência de dados", "outbox pattern", "no redis".

### Alternatives (if observable)
- **Chamada HTTP Síncrona no OrderService**: Descartada explicitamente na reunião por Bruno e Larissa devido ao risco de lentidão de clientes externos travar o processamento de pedidos e inviabilizar rollback limpo.
- **Fila de Mensageria com Redis Streams / RabbitMQ**: Descartada por Diego e Larissa por constituir *overengineering* para a escala atual do time e exigir provisionamento de nova infraestrutura.

## Questions to Address in ADR (if created)

- Qual a estrutura de campos necessária para a tabela `webhook_outbox` (ex: id UUID, event_type, payload JSON renderizado, status, createdAt)?
- Como evitar acúmulo de registros antigos na tabela outbox (política futura de expurgo/arquivamento após 30 dias)?
- A função de publicação deve receber `tx: Prisma.TransactionClient` explicitamente para garantir o acoplamento transacional?

## Related Potential ADRs
- [Worker Desacoplado via Polling](./worker-desacoplado-via-polling.md)
- [Garantia de Entrega At-Least-Once com Desduplicação por Event ID](./garantia-entrega-at-least-once-com-desduplicacao-event-id.md)
- [Banco de Dados Relacional MySQL 8.0](../INFRA/banco-de-dados-relacional-mysql.md)

## Additional Notes
A decisão atende o item 1 do arquivo `docs/adrs/must-include.md` e está classificada obrigatoriamente como `must-document`.
