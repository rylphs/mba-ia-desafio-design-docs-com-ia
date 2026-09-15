# ADR Link Validation Report

**Timestamp:** 2026-09-15-18-45
**Scope:** `docs/adrs/generated/WEBHOOKS` and `docs/adrs`
**Total Links Checked:** 44
**Validation Status:** OK (100% Valid)

## Summary Metrics

- **Total Links Checked:** 44
- **Valid Links:** 44 (100%)
- **Broken Links:** 0
- **Orphaned Links:** 0
- **Reciprocity Checks:** All 11 bidirectional pairs verified successfully in both directories.

## Detailed Checks Log

```
VALID LINK in ADR-001-padrao-transacional-outbox-no-mysql.md: [ADR-002: Worker Desacoplado via Polling](./ADR-002-worker-desacoplado-via-polling.md) -> exists
VALID LINK in ADR-001-padrao-transacional-outbox-no-mysql.md: [ADR-003: Garantia de Entrega At-Least-Once com Desduplicação por Event ID](./ADR-003-garantia-entrega-at-least-once-com-desduplicacao-event-id.md) -> exists
VALID LINK in ADR-001-padrao-transacional-outbox-no-mysql.md: [ADR-004: Política de Retry com Backoff Exponencial e Tabela DLQ Dedicada](./ADR-004-politica-retry-backoff-exponencial-tabela-dlq.md) -> exists
VALID LINK in ADR-001-padrao-transacional-outbox-no-mysql.md: [ADR-005: Autenticação e Integridade via HMAC-SHA256 com Secret por Endpoint](./ADR-005-autenticacao-integridade-hmac-sha256-secret-por-endpoint.md) -> exists
VALID LINK in ADR-001-padrao-transacional-outbox-no-mysql.md: [ADR-006: Reaproveitamento Integral dos Padrões da Codebase](./ADR-006-reaproveitamento-padroes-codebase.md) -> exists
VALID LINK in ADR-002-worker-desacoplado-via-polling.md: [ADR-001: Padrão Transacional Outbox no MySQL](./ADR-001-padrao-transacional-outbox-no-mysql.md) -> exists
VALID LINK in ADR-002-worker-desacoplado-via-polling.md: [ADR-004: Política de Retry com Backoff Exponencial e Tabela DLQ Dedicada](./ADR-004-politica-retry-backoff-exponencial-tabela-dlq.md) -> exists
VALID LINK in ADR-002-worker-desacoplado-via-polling.md: [ADR-006: Reaproveitamento Integral dos Padrões da Codebase](./ADR-006-reaproveitamento-padroes-codebase.md) -> exists
VALID LINK in ADR-003-garantia-entrega-at-least-once-com-desduplicacao-event-id.md: [ADR-001: Padrão Transacional Outbox no MySQL](./ADR-001-padrao-transacional-outbox-no-mysql.md) -> exists
VALID LINK in ADR-003-garantia-entrega-at-least-once-com-desduplicacao-event-id.md: [ADR-004: Política de Retry com Backoff Exponencial e Tabela DLQ Dedicada](./ADR-004-politica-retry-backoff-exponencial-tabela-dlq.md) -> exists
VALID LINK in ADR-003-garantia-entrega-at-least-once-com-desduplicacao-event-id.md: [ADR-005: Autenticação e Integridade via HMAC-SHA256 com Secret por Endpoint](./ADR-005-autenticacao-integridade-hmac-sha256-secret-por-endpoint.md) -> exists
VALID LINK in ADR-004-politica-retry-backoff-exponencial-tabela-dlq.md: [ADR-001: Padrão Transacional Outbox no MySQL](./ADR-001-padrao-transacional-outbox-no-mysql.md) -> exists
VALID LINK in ADR-004-politica-retry-backoff-exponencial-tabela-dlq.md: [ADR-002: Worker Desacoplado via Polling](./ADR-002-worker-desacoplado-via-polling.md) -> exists
VALID LINK in ADR-004-politica-retry-backoff-exponencial-tabela-dlq.md: [ADR-003: Garantia de Entrega At-Least-Once com Desduplicação por Event ID](./ADR-003-garantia-entrega-at-least-once-com-desduplicacao-event-id.md) -> exists
VALID LINK in ADR-004-politica-retry-backoff-exponencial-tabela-dlq.md: [ADR-006: Reaproveitamento Integral dos Padrões da Codebase](./ADR-006-reaproveitamento-padroes-codebase.md) -> exists
VALID LINK in ADR-005-autenticacao-integridade-hmac-sha256-secret-por-endpoint.md: [ADR-001: Padrão Transacional Outbox no MySQL](./ADR-001-padrao-transacional-outbox-no-mysql.md) -> exists
VALID LINK in ADR-005-autenticacao-integridade-hmac-sha256-secret-por-endpoint.md: [ADR-003: Garantia de Entrega At-Least-Once com Desduplicação por Event ID](./ADR-003-garantia-entrega-at-least-once-com-desduplicacao-event-id.md) -> exists
VALID LINK in ADR-005-autenticacao-integridade-hmac-sha256-secret-por-endpoint.md: [ADR-006: Reaproveitamento Integral dos Padrões da Codebase](./ADR-006-reaproveitamento-padroes-codebase.md) -> exists
VALID LINK in ADR-006-reaproveitamento-padroes-codebase.md: [ADR-001: Padrão Transacional Outbox no MySQL](./ADR-001-padrao-transacional-outbox-no-mysql.md) -> exists
VALID LINK in ADR-006-reaproveitamento-padroes-codebase.md: [ADR-002: Worker Desacoplado via Polling](./ADR-002-worker-desacoplado-via-polling.md) -> exists
VALID LINK in ADR-006-reaproveitamento-padroes-codebase.md: [ADR-004: Política de Retry com Backoff Exponencial e Tabela DLQ Dedicada](./ADR-004-politica-retry-backoff-exponencial-tabela-dlq.md) -> exists
VALID LINK in ADR-006-reaproveitamento-padroes-codebase.md: [ADR-005: Autenticação e Integridade via HMAC-SHA256 com Secret por Endpoint](./ADR-005-autenticacao-integridade-hmac-sha256-secret-por-endpoint.md) -> exists
RECIPROCITY OK: ADR-001-padrao-transacional-outbox-no-mysql.md Related to ADR-005-autenticacao-integridade-hmac-sha256-secret-por-endpoint.md <-> ADR-005-autenticacao-integridade-hmac-sha256-secret-por-endpoint.md Related to ADR-001-padrao-transacional-outbox-no-mysql.md
RECIPROCITY OK: ADR-001-padrao-transacional-outbox-no-mysql.md Related to ADR-006-reaproveitamento-padroes-codebase.md <-> ADR-006-reaproveitamento-padroes-codebase.md Related to ADR-001-padrao-transacional-outbox-no-mysql.md
RECIPROCITY OK: ADR-002-worker-desacoplado-via-polling.md Depends on ADR-001-padrao-transacional-outbox-no-mysql.md <-> ADR-001-padrao-transacional-outbox-no-mysql.md Used by ADR-002-worker-desacoplado-via-polling.md
RECIPROCITY OK: ADR-002-worker-desacoplado-via-polling.md Related to ADR-004-politica-retry-backoff-exponencial-tabela-dlq.md <-> ADR-004-politica-retry-backoff-exponencial-tabela-dlq.md Related to ADR-002-worker-desacoplado-via-polling.md
RECIPROCITY OK: ADR-002-worker-desacoplado-via-polling.md Related to ADR-006-reaproveitamento-padroes-codebase.md <-> ADR-006-reaproveitamento-padroes-codebase.md Related to ADR-002-worker-desacoplado-via-polling.md
RECIPROCITY OK: ADR-003-garantia-entrega-at-least-once-com-desduplicacao-event-id.md Depends on ADR-001-padrao-transacional-outbox-no-mysql.md <-> ADR-001-padrao-transacional-outbox-no-mysql.md Used by ADR-003-garantia-entrega-at-least-once-com-desduplicacao-event-id.md
RECIPROCITY OK: ADR-003-garantia-entrega-at-least-once-com-desduplicacao-event-id.md Related to ADR-004-politica-retry-backoff-exponencial-tabela-dlq.md <-> ADR-004-politica-retry-backoff-exponencial-tabela-dlq.md Related to ADR-003-garantia-entrega-at-least-once-com-desduplicacao-event-id.md
RECIPROCITY OK: ADR-003-garantia-entrega-at-least-once-com-desduplicacao-event-id.md Related to ADR-005-autenticacao-integridade-hmac-sha256-secret-por-endpoint.md <-> ADR-005-autenticacao-integridade-hmac-sha256-secret-por-endpoint.md Related to ADR-003-garantia-entrega-at-least-once-com-desduplicacao-event-id.md
RECIPROCITY OK: ADR-004-politica-retry-backoff-exponencial-tabela-dlq.md Depends on ADR-001-padrao-transacional-outbox-no-mysql.md <-> ADR-001-padrao-transacional-outbox-no-mysql.md Used by ADR-004-politica-retry-backoff-exponencial-tabela-dlq.md
RECIPROCITY OK: ADR-004-politica-retry-backoff-exponencial-tabela-dlq.md Related to ADR-002-worker-desacoplado-via-polling.md <-> ADR-002-worker-desacoplado-via-polling.md Related to ADR-004-politica-retry-backoff-exponencial-tabela-dlq.md
RECIPROCITY OK: ADR-004-politica-retry-backoff-exponencial-tabela-dlq.md Related to ADR-003-garantia-entrega-at-least-once-com-desduplicacao-event-id.md <-> ADR-003-garantia-entrega-at-least-once-com-desduplicacao-event-id.md Related to ADR-004-politica-retry-backoff-exponencial-tabela-dlq.md
RECIPROCITY OK: ADR-004-politica-retry-backoff-exponencial-tabela-dlq.md Related to ADR-006-reaproveitamento-padroes-codebase.md <-> ADR-006-reaproveitamento-padroes-codebase.md Related to ADR-004-politica-retry-backoff-exponencial-tabela-dlq.md
RECIPROCITY OK: ADR-005-autenticacao-integridade-hmac-sha256-secret-por-endpoint.md Related to ADR-001-padrao-transacional-outbox-no-mysql.md <-> ADR-001-padrao-transacional-outbox-no-mysql.md Related to ADR-005-autenticacao-integridade-hmac-sha256-secret-por-endpoint.md
RECIPROCITY OK: ADR-005-autenticacao-integridade-hmac-sha256-secret-por-endpoint.md Related to ADR-003-garantia-entrega-at-least-once-com-desduplicacao-event-id.md <-> ADR-003-garantia-entrega-at-least-once-com-desduplicacao-event-id.md Related to ADR-005-autenticacao-integridade-hmac-sha256-secret-por-endpoint.md
RECIPROCITY OK: ADR-005-autenticacao-integridade-hmac-sha256-secret-por-endpoint.md Related to ADR-006-reaproveitamento-padroes-codebase.md <-> ADR-006-reaproveitamento-padroes-codebase.md Related to ADR-005-autenticacao-integridade-hmac-sha256-secret-por-endpoint.md
RECIPROCITY OK: ADR-006-reaproveitamento-padroes-codebase.md Related to ADR-001-padrao-transacional-outbox-no-mysql.md <-> ADR-001-padrao-transacional-outbox-no-mysql.md Related to ADR-006-reaproveitamento-padroes-codebase.md
RECIPROCITY OK: ADR-006-reaproveitamento-padroes-codebase.md Related to ADR-002-worker-desacoplado-via-polling.md <-> ADR-002-worker-desacoplado-via-polling.md Related to ADR-006-reaproveitamento-padroes-codebase.md
RECIPROCITY OK: ADR-006-reaproveitamento-padroes-codebase.md Related to ADR-004-politica-retry-backoff-exponencial-tabela-dlq.md <-> ADR-004-politica-retry-backoff-exponencial-tabela-dlq.md Related to ADR-006-reaproveitamento-padroes-codebase.md
RECIPROCITY OK: ADR-006-reaproveitamento-padroes-codebase.md Related to ADR-005-autenticacao-integridade-hmac-sha256-secret-por-endpoint.md <-> ADR-005-autenticacao-integridade-hmac-sha256-secret-por-endpoint.md Related to ADR-006-reaproveitamento-padroes-codebase.md
VALID LINK in ADR-001-padrao-transacional-outbox-no-mysql.md: [ADR-002: Worker Desacoplado via Polling](./ADR-002-worker-desacoplado-via-polling.md) -> exists
VALID LINK in ADR-001-padrao-transacional-outbox-no-mysql.md: [ADR-003: Garantia de Entrega At-Least-Once com Desduplicação por Event ID](./ADR-003-garantia-entrega-at-least-once-com-desduplicacao-event-id.md) -> exists
VALID LINK in ADR-001-padrao-transacional-outbox-no-mysql.md: [ADR-004: Política de Retry com Backoff Exponencial e Tabela DLQ Dedicada](./ADR-004-politica-retry-backoff-exponencial-tabela-dlq.md) -> exists
VALID LINK in ADR-001-padrao-transacional-outbox-no-mysql.md: [ADR-005: Autenticação e Integridade via HMAC-SHA256 com Secret por Endpoint](./ADR-005-autenticacao-integridade-hmac-sha256-secret-por-endpoint.md) -> exists
VALID LINK in ADR-001-padrao-transacional-outbox-no-mysql.md: [ADR-006: Reaproveitamento Integral dos Padrões da Codebase](./ADR-006-reaproveitamento-padroes-codebase.md) -> exists
VALID LINK in ADR-002-worker-desacoplado-via-polling.md: [ADR-001: Padrão Transacional Outbox no MySQL](./ADR-001-padrao-transacional-outbox-no-mysql.md) -> exists
VALID LINK in ADR-002-worker-desacoplado-via-polling.md: [ADR-004: Política de Retry com Backoff Exponencial e Tabela DLQ Dedicada](./ADR-004-politica-retry-backoff-exponencial-tabela-dlq.md) -> exists
VALID LINK in ADR-002-worker-desacoplado-via-polling.md: [ADR-006: Reaproveitamento Integral dos Padrões da Codebase](./ADR-006-reaproveitamento-padroes-codebase.md) -> exists
VALID LINK in ADR-003-garantia-entrega-at-least-once-com-desduplicacao-event-id.md: [ADR-001: Padrão Transacional Outbox no MySQL](./ADR-001-padrao-transacional-outbox-no-mysql.md) -> exists
VALID LINK in ADR-003-garantia-entrega-at-least-once-com-desduplicacao-event-id.md: [ADR-004: Política de Retry com Backoff Exponencial e Tabela DLQ Dedicada](./ADR-004-politica-retry-backoff-exponencial-tabela-dlq.md) -> exists
VALID LINK in ADR-003-garantia-entrega-at-least-once-com-desduplicacao-event-id.md: [ADR-005: Autenticação e Integridade via HMAC-SHA256 com Secret por Endpoint](./ADR-005-autenticacao-integridade-hmac-sha256-secret-por-endpoint.md) -> exists
VALID LINK in ADR-004-politica-retry-backoff-exponencial-tabela-dlq.md: [ADR-001: Padrão Transacional Outbox no MySQL](./ADR-001-padrao-transacional-outbox-no-mysql.md) -> exists
VALID LINK in ADR-004-politica-retry-backoff-exponencial-tabela-dlq.md: [ADR-002: Worker Desacoplado via Polling](./ADR-002-worker-desacoplado-via-polling.md) -> exists
VALID LINK in ADR-004-politica-retry-backoff-exponencial-tabela-dlq.md: [ADR-003: Garantia de Entrega At-Least-Once com Desduplicação por Event ID](./ADR-003-garantia-entrega-at-least-once-com-desduplicacao-event-id.md) -> exists
VALID LINK in ADR-004-politica-retry-backoff-exponencial-tabela-dlq.md: [ADR-006: Reaproveitamento Integral dos Padrões da Codebase](./ADR-006-reaproveitamento-padroes-codebase.md) -> exists
VALID LINK in ADR-005-autenticacao-integridade-hmac-sha256-secret-por-endpoint.md: [ADR-001: Padrão Transacional Outbox no MySQL](./ADR-001-padrao-transacional-outbox-no-mysql.md) -> exists
VALID LINK in ADR-005-autenticacao-integridade-hmac-sha256-secret-por-endpoint.md: [ADR-003: Garantia de Entrega At-Least-Once com Desduplicação por Event ID](./ADR-003-garantia-entrega-at-least-once-com-desduplicacao-event-id.md) -> exists
VALID LINK in ADR-005-autenticacao-integridade-hmac-sha256-secret-por-endpoint.md: [ADR-006: Reaproveitamento Integral dos Padrões da Codebase](./ADR-006-reaproveitamento-padroes-codebase.md) -> exists
VALID LINK in ADR-006-reaproveitamento-padroes-codebase.md: [ADR-001: Padrão Transacional Outbox no MySQL](./ADR-001-padrao-transacional-outbox-no-mysql.md) -> exists
VALID LINK in ADR-006-reaproveitamento-padroes-codebase.md: [ADR-002: Worker Desacoplado via Polling](./ADR-002-worker-desacoplado-via-polling.md) -> exists
VALID LINK in ADR-006-reaproveitamento-padroes-codebase.md: [ADR-004: Política de Retry com Backoff Exponencial e Tabela DLQ Dedicada](./ADR-004-politica-retry-backoff-exponencial-tabela-dlq.md) -> exists
VALID LINK in ADR-006-reaproveitamento-padroes-codebase.md: [ADR-005: Autenticação e Integridade via HMAC-SHA256 com Secret por Endpoint](./ADR-005-autenticacao-integridade-hmac-sha256-secret-por-endpoint.md) -> exists
RECIPROCITY OK: ADR-001-padrao-transacional-outbox-no-mysql.md Related to ADR-005-autenticacao-integridade-hmac-sha256-secret-por-endpoint.md <-> ADR-005-autenticacao-integridade-hmac-sha256-secret-por-endpoint.md Related to ADR-001-padrao-transacional-outbox-no-mysql.md
RECIPROCITY OK: ADR-001-padrao-transacional-outbox-no-mysql.md Related to ADR-006-reaproveitamento-padroes-codebase.md <-> ADR-006-reaproveitamento-padroes-codebase.md Related to ADR-001-padrao-transacional-outbox-no-mysql.md
RECIPROCITY OK: ADR-002-worker-desacoplado-via-polling.md Depends on ADR-001-padrao-transacional-outbox-no-mysql.md <-> ADR-001-padrao-transacional-outbox-no-mysql.md Used by ADR-002-worker-desacoplado-via-polling.md
RECIPROCITY OK: ADR-002-worker-desacoplado-via-polling.md Related to ADR-004-politica-retry-backoff-exponencial-tabela-dlq.md <-> ADR-004-politica-retry-backoff-exponencial-tabela-dlq.md Related to ADR-002-worker-desacoplado-via-polling.md
RECIPROCITY OK: ADR-002-worker-desacoplado-via-polling.md Related to ADR-006-reaproveitamento-padroes-codebase.md <-> ADR-006-reaproveitamento-padroes-codebase.md Related to ADR-002-worker-desacoplado-via-polling.md
RECIPROCITY OK: ADR-003-garantia-entrega-at-least-once-com-desduplicacao-event-id.md Depends on ADR-001-padrao-transacional-outbox-no-mysql.md <-> ADR-001-padrao-transacional-outbox-no-mysql.md Used by ADR-003-garantia-entrega-at-least-once-com-desduplicacao-event-id.md
RECIPROCITY OK: ADR-003-garantia-entrega-at-least-once-com-desduplicacao-event-id.md Related to ADR-004-politica-retry-backoff-exponencial-tabela-dlq.md <-> ADR-004-politica-retry-backoff-exponencial-tabela-dlq.md Related to ADR-003-garantia-entrega-at-least-once-com-desduplicacao-event-id.md
RECIPROCITY OK: ADR-003-garantia-entrega-at-least-once-com-desduplicacao-event-id.md Related to ADR-005-autenticacao-integridade-hmac-sha256-secret-por-endpoint.md <-> ADR-005-autenticacao-integridade-hmac-sha256-secret-por-endpoint.md Related to ADR-003-garantia-entrega-at-least-once-com-desduplicacao-event-id.md
RECIPROCITY OK: ADR-004-politica-retry-backoff-exponencial-tabela-dlq.md Depends on ADR-001-padrao-transacional-outbox-no-mysql.md <-> ADR-001-padrao-transacional-outbox-no-mysql.md Used by ADR-004-politica-retry-backoff-exponencial-tabela-dlq.md
RECIPROCITY OK: ADR-004-politica-retry-backoff-exponencial-tabela-dlq.md Related to ADR-002-worker-desacoplado-via-polling.md <-> ADR-002-worker-desacoplado-via-polling.md Related to ADR-004-politica-retry-backoff-exponencial-tabela-dlq.md
RECIPROCITY OK: ADR-004-politica-retry-backoff-exponencial-tabela-dlq.md Related to ADR-003-garantia-entrega-at-least-once-com-desduplicacao-event-id.md <-> ADR-003-garantia-entrega-at-least-once-com-desduplicacao-event-id.md Related to ADR-004-politica-retry-backoff-exponencial-tabela-dlq.md
RECIPROCITY OK: ADR-004-politica-retry-backoff-exponencial-tabela-dlq.md Related to ADR-006-reaproveitamento-padroes-codebase.md <-> ADR-006-reaproveitamento-padroes-codebase.md Related to ADR-004-politica-retry-backoff-exponencial-tabela-dlq.md
RECIPROCITY OK: ADR-005-autenticacao-integridade-hmac-sha256-secret-por-endpoint.md Related to ADR-001-padrao-transacional-outbox-no-mysql.md <-> ADR-001-padrao-transacional-outbox-no-mysql.md Related to ADR-005-autenticacao-integridade-hmac-sha256-secret-por-endpoint.md
RECIPROCITY OK: ADR-005-autenticacao-integridade-hmac-sha256-secret-por-endpoint.md Related to ADR-003-garantia-entrega-at-least-once-com-desduplicacao-event-id.md <-> ADR-003-garantia-entrega-at-least-once-com-desduplicacao-event-id.md Related to ADR-005-autenticacao-integridade-hmac-sha256-secret-por-endpoint.md
RECIPROCITY OK: ADR-005-autenticacao-integridade-hmac-sha256-secret-por-endpoint.md Related to ADR-006-reaproveitamento-padroes-codebase.md <-> ADR-006-reaproveitamento-padroes-codebase.md Related to ADR-005-autenticacao-integridade-hmac-sha256-secret-por-endpoint.md
RECIPROCITY OK: ADR-006-reaproveitamento-padroes-codebase.md Related to ADR-001-padrao-transacional-outbox-no-mysql.md <-> ADR-001-padrao-transacional-outbox-no-mysql.md Related to ADR-006-reaproveitamento-padroes-codebase.md
RECIPROCITY OK: ADR-006-reaproveitamento-padroes-codebase.md Related to ADR-002-worker-desacoplado-via-polling.md <-> ADR-002-worker-desacoplado-via-polling.md Related to ADR-006-reaproveitamento-padroes-codebase.md
RECIPROCITY OK: ADR-006-reaproveitamento-padroes-codebase.md Related to ADR-004-politica-retry-backoff-exponencial-tabela-dlq.md <-> ADR-004-politica-retry-backoff-exponencial-tabela-dlq.md Related to ADR-006-reaproveitamento-padroes-codebase.md
RECIPROCITY OK: ADR-006-reaproveitamento-padroes-codebase.md Related to ADR-005-autenticacao-integridade-hmac-sha256-secret-por-endpoint.md <-> ADR-005-autenticacao-integridade-hmac-sha256-secret-por-endpoint.md Related to ADR-006-reaproveitamento-padroes-codebase.md
```

## Result: PASSED
