# Potential ADR: Reaproveitamento Integral dos Padrões da Codebase

**Module**: WEBHOOKS
**Category**: Architecture
**Priority**: Must Document (Score: 135 / Override: docs/adrs/must-include.md)
**Date Identified**: 2026-09-13

---

## Existing ADR Context

ℹ️ **RELATED DECISIONS**

This decision relates to:
- **Framework Web Express.js** (`INFRA`): Estrutura de roteamento e middlewares existente.
- **ORM Prisma** (`INFRA`): Mecanismo de persistência de dados utilizado no novo módulo.
- **Arquitetura de API REST com Tratamento Centralizado de Erros** (`INFRA`): Reuso de `AppError` e middleware centralizado.
- **Autenticação Stateless com JWT e RBAC** (`AUTH`): Reuso de `requireRole` para permissões administrativas.

**When creating formal ADR**: Reference these in Related ADRs section.

---

## What Was Identified

Identificou-se a decisão técnica de implementar a nova funcionalidade de Webhooks aderindo estritamente aos padrões arquiteturais, estruturais e de convenção de código já consolidados na codebase existente do OMS.

A decisão proíbe a introdução de novos paradigmas concorrentes ou bibliotecas externas não justificadas, determinando:
1. **Estrutura Modular Homogênea**: Criação de `src/modules/webhooks/` contendo `webhook.controller.ts`, `webhook.service.ts`, `webhook.repository.ts`, `webhook.routes.ts` e `webhook.schemas.ts`, mantendo a separação em camadas adotada em `orders`, `products`, `customers`, `users` e `auth`.
2. **Tratamento de Erros Padronizado**: Utilização da hierarquia baseada em `AppError`, com classes especializadas e códigos de erro de máquina estritamente prefixados com `WEBHOOK_*` (ex: `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`, `WEBHOOK_SECRET_REQUIRED`). O middleware global existente em `src/middlewares/error.middleware.ts` tratará as exceções sem modificações.
3. **Validação e Contratos com Zod**: Validação estrita de parâmetros de rota, query strings e bodies JSON através de esquemas Zod integrados via middleware de validação existente.
4. **Segurança e RBAC Existente**: Reutilização dos middlewares `authenticate` e `requireRole('ADMIN')` já existentes em `src/middlewares/auth.middleware.ts` para autorização das rotas do módulo, especificamente para proteção de replays da DLQ.
5. **Observabilidade e Logs Estruturados com Pino**: Uso do logger instanciado em `src/shared/logger/index.ts` em todo o ciclo de vida do worker e dos controllers, sem introduzir ferramentas adicionais de log.

## Why This Might Deserve an ADR

- **Impact**: Mantém a homogeneidade e manutenibilidade de todo o repositório, garantindo que qualquer desenvolvedor familiarizado com os módulos existentes consiga manter, depurar e evoluir o módulo de webhooks sem curva de aprendizado adicional.
- **Trade-offs**: Limita a liberdade técnica de adotar frameworks ou abordagens experimentais no novo módulo em favor da coesão e previsibilidade de longo prazo do sistema.
- **Complexity**: Baixa. Acelerada pelo reuso direto de componentes já testados e validados.
- **Team Knowledge**: Alto benefício. O time reaproveita 100% de seu conhecimento prático sobre o fluxo de requisições e tratamento de erros.
- **Future Implications**: Estabelece uma cultura de integridade e consistência arquitetural na expansão de novas features no produto.

## Evidence Found in Codebase

### Key Files
- [`src/shared/errors/app-error.ts`](file:///home/raphael/workspace/pos-ia/desafios/mba-ia-desafio-design-docs-com-ia/src/shared/errors/app-error.ts#L1-L17) - Linhas 1-17
  - Definição da classe base `AppError`.
- [`src/middlewares/error.middleware.ts`](file:///home/raphael/workspace/pos-ia/desafios/mba-ia-desafio-design-docs-com-ia/src/middlewares/error.middleware.ts#L14-L65) - Linhas 14-65
  - Middleware centralizado que intercepta `AppError`, `ZodError` e erros conhecidos do Prisma.
- [`src/shared/logger/index.ts`](file:///home/raphael/workspace/pos-ia/desafios/mba-ia-desafio-design-docs-com-ia/src/shared/logger/index.ts)
  - Logger estruturado baseado em Pino.
- [`src/middlewares/auth.middleware.ts`](file:///home/raphael/workspace/pos-ia/desafios/mba-ia-desafio-design-docs-com-ia/src/middlewares/auth.middleware.ts#L49-L62) - Linhas 49-62
  - Middleware `requireRole` para autorização de perfis.
- [`TRANSCRICAO.md`](file:///home/raphael/workspace/pos-ia/desafios/mba-ia-desafio-design-docs-com-ia/TRANSCRICAO.md#L160-L181) - Linhas 160-181
  - Bruno, Diego e Larissa acordando o reuso máximo de `AppError`, prefixo `WEBHOOK_`, middleware de erro e logger Pino.
- [`docs/adrs/must-include.md`](file:///home/raphael/workspace/pos-ia/desafios/mba-ia-desafio-design-docs-com-ia/docs/adrs/must-include.md#L10) - Linha 10
  - Requisito mandatório de documentação técnica.

### Code Evidence
```typescript
// Exemplo em src/shared/errors/app-error.ts:3-16
export class AppError extends Error {
  public readonly statusCode: number;
  public readonly errorCode: string;
  public readonly details: ErrorDetails;

  constructor(message: string, statusCode: number, errorCode: string, details?: ErrorDetails) {
    super(message);
    this.name = 'AppError';
    this.statusCode = statusCode;
    this.errorCode = errorCode;
    this.details = details;
    Error.captureStackTrace?.(this, this.constructor);
  }
}
```

### Impact Analysis
- Introduced: Fundamentado no design original da aplicação (`7ef4317`) e reafirmado para a feature de webhooks em 2026-09-13.
- Modified: Consistente em todos os módulos da aplicação (`auth`, `users`, `customers`, `products`, `orders`).
- Last change: 2026-09-13.
- Affects: Toda a implementação do módulo `src/modules/webhooks/`, integrando-se nativamente à arquitetura Express e Prisma.
- Recent themes: "consistência de código", "AppError", "prefixo WEBHOOK_", "reuso de middlewares".

### Alternatives (if observable)
- **Construir Webhooks como Microserviço Independente em Outra Linguagem/Stack**: Descartado por Diego e Larissa para evitar fragmentação de stack, pipelines de CI/CD redundantes e custos adicionais de infraestrutura para um time enxuto.
- **Tratamento Ad-hoc de Erros ou Novas Bibliotecas de Log (Winston/Morgan)**: Descartado por Bruno e Larissa pela redundância desnecessária.

## Questions to Address in ADR (if created)

- Quais são todas as classes de erro específicas do módulo de webhooks a serem criadas herdando de `AppError`?
- Como o router de webhooks será registrado em `src/app.ts` (`buildControllers` e `buildApiRouter`)?

## Related Potential ADRs
- [Padrão Transacional Outbox no MySQL](./padrao-transacional-outbox-no-mysql.md)
- [Arquitetura de API REST com Tratamento Centralizado de Erros](../INFRA/arquitetura-api-rest.md)
- [Autenticação Stateless com JWT e RBAC](../AUTH/autenticacao-jwt-e-rbac.md)

## Additional Notes
A decisão atende o item 6 do arquivo `docs/adrs/must-include.md` e está classificada obrigatoriamente como `must-document`.
