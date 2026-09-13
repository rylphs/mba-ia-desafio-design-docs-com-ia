# Potential ADR: Política de Retry com Backoff Exponencial e Tabela DLQ Dedicada

**Module**: WEBHOOKS
**Category**: Architecture
**Priority**: Must Document (Score: 145 / Override: docs/adrs/must-include.md)
**Date Identified**: 2026-09-13

---

## Existing ADR Context

ℹ️ **RELATED DECISIONS**

This decision relates to:
- **Worker Desacoplado via Polling** (`WEBHOOKS`): O componente responsável por avaliar tentativas e aplicar o cálculo de backoff.
- **Padrão Transacional Outbox no MySQL** (`WEBHOOKS`): De onde os eventos com falha definitiva são transferidos para a tabela DLQ.
- **Reaproveitamento Integral dos Padrões da Codebase** (`WEBHOOKS`): Reuso de autenticação e RBAC (`requireRole('ADMIN')`) no endpoint de replay.

**When creating formal ADR**: Reference these in Related ADRs section.

---

## What Was Identified

Identificou-se a decisão de adotar uma política estruturada de tolerância a falhas composta por retentativas automáticas com *Backoff Exponencial*, limitada a 5 tentativas espaçadas, e encaminhamento de eventos com falha definitiva para uma tabela dedicada de *Dead Letter Queue* (`webhook_dead_letter`), suportando reprocessamento manual via endpoint administrativo autenticado com perfil `ADMIN` (`POST /admin/webhooks/dead-letter/:id/replay`).

A progressão dos intervalos de retentativa foi definida em:
1. 1ª retentativa: após 1 minuto
2. 2ª retentativa: após 5 minutos
3. 3ª retentativa: após 30 minutos
4. 4ª retentativa: após 2 horas
5. 5ª retentativa: após 12 horas

Essa janela cobre um total aproximado de 15 horas desde a primeira falha até a última tentativa, acomodando manutenções planejadas ou indisponibilidades prolongadas na infraestrutura dos clientes B2B sem sobrecarregar seus servidores com disparos contínuos. Caso a 5ª tentativa falhe ou ocorra um erro irrecuperável, o evento é movido para `webhook_dead_letter`, contendo o payload original, a causa do erro, o status HTTP retornado e timestamp detalhado.

## Why This Might Deserve an ADR

- **Impact**: Garante alta resiliência e observabilidade sobre falhas de integração com clientes externos, evitando que falhas temporárias resultem em perda permanente de mensagens ou bloqueiem o envio de eventos para outros clientes.
- **Trade-offs**: A separação física em uma tabela dedicada de DLQ isola o tráfego da outbox principal, mas requer a criação e manutenção de endpoints administrativos adicionais e governança operacional para monitoramento da DLQ.
- **Complexity**: Média. Envolve cálculo de `nextRetryAt` baseado no número da tentativa (`attemptCount`), gravação atômica na tabela DLQ e implementação do endpoint seguro de replay.
- **Team Knowledge**: Alto. Engenheiros de suporte e operações precisam saber como investigar falhas na DLQ e acionar o replay manual após a resolução de problemas no cliente.
- **Future Implications**: Permite a criação futura de alertas de monitoramento quando a tabela DLQ receber novos registros, sem degradar a tabela de outbox.

## Evidence Found in Codebase

### Key Files
- [`TRANSCRICAO.md`](file:///home/raphael/workspace/pos-ia/desafios/mba-ia-desafio-design-docs-com-ia/TRANSCRICAO.md#L90-L116) - Linhas 90-116 e 204-214
  - Diego, Bruno e Larissa discutindo e descartando 3 tentativas por ser pouco e aprovando 5 tentativas (1m, 5m, 30m, 2h, 12h) com tabela DLQ dedicada e endpoint admin de replay.
- [`docs/adrs/must-include.md`](file:///home/raphael/workspace/pos-ia/desafios/mba-ia-desafio-design-docs-com-ia/docs/adrs/must-include.md#L8) - Linha 8
  - Requisito mandatório de documentação técnica.
- [`src/middlewares/auth.middleware.ts`](file:///home/raphael/workspace/pos-ia/desafios/mba-ia-desafio-design-docs-com-ia/src/middlewares/auth.middleware.ts#L49-L62) - Linhas 49-62
  - Middleware `requireRole('ADMIN')` que protegerá o endpoint `POST /admin/webhooks/dead-letter/:id/replay`.

### Code Evidence
```typescript
// Progressão de backoff exponencial e cálculo de próxima tentativa:
const RETRY_DELAYS_MS = [
  1 * 60 * 1000,        // 1m  (tentativa 1)
  5 * 60 * 1000,        // 5m  (tentativa 2)
  30 * 60 * 1000,       // 30m (tentativa 3)
  2 * 60 * 60 * 1000,   // 2h  (tentativa 4)
  12 * 60 * 60 * 1000,  // 12h (tentativa 5)
];

// Endpoint administrativo de replay protegido por RBAC:
// router.post(
//   '/admin/webhooks/dead-letter/:id/replay',
//   authenticate,
//   requireRole('ADMIN'),
//   controller.replayDeadLetter
// );
```

### Impact Analysis
- Introduced: Especificado na reunião de 2026-09-13.
- Modified: Integrado aos fluxos de resiliência e tolerância a falhas.
- Last change: 2026-09-13.
- Affects: Worker de processamento, esquema Prisma (`webhook_dead_letter`), rotas de administração e auditoria.
- Recent themes: "backoff exponencial", "DLQ dedicada", "5 tentativas", "15 horas de janela", "replay admin".

### Alternatives (if observable)
- **Retry Curto e Fixo (3 tentativas em 30 minutos)**: Descartado por Diego e Bruno porque clientes realizam janelas de manutenção de várias horas, o que levaria à perda prematura de notificações.
- **Retry Indefinito**: Descartado por Diego pelo risco de acumular eventos pendurados eternamente para clientes que desativaram ou abandonaram seus endpoints.
- **Sinalizar Falha na Própria Tabela Outbox (sem DLQ separada)**: Descartado por Diego para não poluir a tabela de outbox que é consultada a cada 2 segundos pelo worker.

## Questions to Address in ADR (if created)

- O que acontece quando o replay manual de um evento na DLQ é disparado (ele volta para a outbox como pendente com `attempts = 0`)?
- Como deve ser estruturado o log de auditoria no replay (gravar ID do usuário ADMIN executor da ação)?
- Deve existir limite temporal para permanência de itens na tabela de DLQ (ex: 90 dias)?

## Related Potential ADRs
- [Worker Desacoplado via Polling](./worker-desacoplado-via-polling.md)
- [Padrão Transacional Outbox no MySQL](./padrao-transacional-outbox-no-mysql.md)
- [Reaproveitamento Integral dos Padrões da Codebase](./reaproveitamento-padroes-codebase.md)

## Additional Notes
A decisão atende o item 4 do arquivo `docs/adrs/must-include.md` e está classificada obrigatoriamente como `must-document`.
