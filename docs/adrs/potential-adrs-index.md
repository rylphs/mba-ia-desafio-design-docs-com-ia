# Potential ADRs Index

## Analysis Progress

### Analyzed Modules
- **WEBHOOKS**: Sistema de Webhooks e Notificações - 2026-09-13 - 6 high, 0 medium ADRs
- **INFRA**: Infraestrutura, Persistência e Transversais - 2026-09-13 - 4 high, 1 medium ADRs
- **AUTH**: Autenticação e Autorização - 2026-09-13 - 1 high, 0 medium ADRs
- **ORDERS**: Pedidos e Ciclo de Vida - 2026-09-13 - 0 high, 0 medium ADRs (decisões mapeadas em WEBHOOKS e INFRA)
- **USERS**: Gestão de Usuários - 2026-09-13 - 0 high, 0 medium ADRs (decisões mapeadas em AUTH e INFRA)
- **CUSTOMERS**: Gestão de Clientes - 2026-09-13 - 0 high, 0 medium ADRs
- **PRODUCTS**: Catálogo e Estoque - 2026-09-13 - 0 high, 0 medium ADRs

### Pending Analysis
- Nenhuma pendência. Todos os 7 módulos identificados no mapeamento arquitetural foram analisados.

---

## High Priority ADRs (must-document/)

> [!NOTE]
> Conforme instrução expressa de projeto, todos os 6 itens contidos no arquivo `docs/adrs/must-include.md` foram categorizados obrigatoriamente como `must-document`, independentemente do cálculo do score. As demais decisões foram submetidas ao processo padrão de pontuação (Step 0 - Decisões Estruturais Universais e Infraestrutura de Domínio).

### Module: WEBHOOKS
| Title | Category | File |
|-------|----------|------|
| Padrão Transacional Outbox no MySQL | Architecture | [Link](./potential-adrs/done/WEBHOOKS/padrao-transacional-outbox-no-mysql.md) |
| Worker Desacoplado via Polling | Architecture | [Link](./potential-adrs/done/WEBHOOKS/worker-desacoplado-via-polling.md) |
| Garantia de Entrega At-Least-Once com Desduplicação por Event ID | Architecture | [Link](./potential-adrs/done/WEBHOOKS/garantia-entrega-at-least-once-com-desduplicacao-event-id.md) |
| Política de Retry com Backoff Exponencial e Tabela DLQ Dedicada | Architecture | [Link](./potential-adrs/done/WEBHOOKS/politica-retry-backoff-exponencial-tabela-dlq.md) |
| Autenticação e Integridade via HMAC-SHA256 com Secret por Endpoint | Security | [Link](./potential-adrs/done/WEBHOOKS/autenticacao-integridade-hmac-sha256-secret-por-endpoint.md) |
| Reaproveitamento Integral dos Padrões da Codebase | Architecture | [Link](./potential-adrs/done/WEBHOOKS/reaproveitamento-padroes-codebase.md) |

### Module: INFRA
| Title | Category | File |
|-------|----------|------|
| Banco de Dados Relacional MySQL 8.0 como Armazenamento Primário | Technology | [Link](./potential-adrs/must-document/INFRA/banco-de-dados-relacional-mysql.md) |
| Framework Web Express.js para Construção da API REST | Platform | [Link](./potential-adrs/must-document/INFRA/framework-web-express.md) |
| ORM Prisma para Acesso a Dados e Modelagem de Esquema | Data Access | [Link](./potential-adrs/must-document/INFRA/orm-prisma.md) |
| Arquitetura de API REST com Tratamento Centralizado de Erros | Architecture | [Link](./potential-adrs/must-document/INFRA/arquitetura-api-rest.md) |

### Module: AUTH
| Title | Category | File |
|-------|----------|------|
| Autenticação Stateless com JWT e Controle de Acesso Baseado em Papéis (RBAC) | Security | [Link](./potential-adrs/must-document/AUTH/autenticacao-jwt-e-rbac.md) |

---

## Medium Priority ADRs (consider/)

### Module: INFRA
| Title | Category | File |
|-------|----------|------|
| Estratégia de Identificadores UUID v4 como Chave Primária | Data Modeling | [Link](./potential-adrs/consider/INFRA/estrategia-identificadores-uuid.md) |

---

## Summary

- **High Priority (`must-document/`)**: 11 ADRs
  - 6 decisões do Sistema de Webhooks (validadas por `docs/adrs/must-include.md` e `TRANSCRICAO.md`)
  - 4 decisões estruturais universais da Codebase (MySQL 8.0, Express.js, Prisma ORM, REST API)
  - 1 decisão de infraestrutura de domínio de segurança (Autenticação JWT e RBAC)
- **Medium Priority (`consider/`)**: 1 ADR (Estratégia de Identificadores UUID v4)
- **Total de Potenciais ADRs Identificadas**: 12 ADRs
- **Módulos Analisados**: 7 de 7
