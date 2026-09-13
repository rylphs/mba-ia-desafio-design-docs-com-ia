# Potential ADR: Arquitetura de API REST com Tratamento Centralizado de Erros

**Module**: INFRA
**Category**: Architecture / API
**Priority**: Must Document (Score: 135)
**Date Identified**: 2026-09-13

---

## Existing ADR Context

ℹ️ **RELATED DECISIONS**

This decision relates to:
- **Framework Web Express.js** (`INFRA`): Base tecnológica sobre a qual a API REST opera.
- **Reaproveitamento Integral dos Padrões da Codebase** (`WEBHOOKS`): Reuso do padrão de rotas REST, contratos e respostas JSON.

**When creating formal ADR**: Reference these in Related ADRs section.

---

## What Was Identified

Identificou-se a decisão de estruturar a comunicação síncrona externa e interna do OMS através de uma **Arquitetura de API RESTful** sobre HTTP/JSON, com convenções rígidas de status codes HTTP, versionamento por path prefix (`/api/v1`) e um mecanismo centralizado e determinístico de tratamento e serialização de erros.

A API adota:
1. **Padrão de Resposta JSON Padronizado**: Estrutura consistente para retornos de sucesso (`data`), paginação uniforme (`meta: { page, pageSize, total, totalPages }`) via helper [`paginated`](file:///home/raphael/workspace/pos-ia/desafios/mba-ia-desafio-design-docs-com-ia/src/shared/http/response.ts#L8-L23) e payload de erro estruturado (`error: { code, message, details }`).
2. **Hierarquia Tipada de Erros de Domínio**: Baseada em [`AppError`](file:///home/raphael/workspace/pos-ia/desafios/mba-ia-desafio-design-docs-com-ia/src/shared/errors/app-error.ts), estendida por erros específicos de semântica HTTP (`NotFoundError`, `ConflictError`, `ValidationError`, `UnauthorizedError`, `ForbiddenError`, `UnprocessableEntityError`) e de negócio (`InsufficientStockError`, `InvalidStatusTransitionError`).
3. **Middleware Global Interceptor**: Implementado em [`src/middlewares/error.middleware.ts`](file:///home/raphael/workspace/pos-ia/desafios/mba-ia-desafio-design-docs-com-ia/src/middlewares/error.middleware.ts), capturando de maneira transparente instâncias de `AppError`, falhas de validação de esquemas `ZodError` e violações de constraints de banco do Prisma (`P2002` conflito, `P2025` não encontrado), além de mascarar erros inesperados (500) com log estruturado e ID de correlação (`requestId`).

## Why This Might Deserve an ADR

- **Impact**: Define os contratos públicos de comunicação entre o OMS e todos os clientes integradores (frontends, ERPs de parceiros B2B e sistemas terceiros).
- **Trade-offs**: A arquitetura REST sobre JSON é universal e intuitiva para desenvolvedores externos, mas impõe sobrecarga de serialização e falta de flexibilidade de seleção de campos dinâmicos inerentes a abordagens como GraphQL.
- **Complexity**: Baixa a média. A centralização do middleware de erro desonera os controllers de blocos repetitivos de try/catch manuais de formatação.
- **Team Knowledge**: Alto. Engenheiros precisam apenas disparar exceções semânticas herdadas de `AppError` dentro de services ou repositórios, sabendo que a resposta HTTP adequada será gerada.
- **Future Implications**: Novos módulos (como o CRUD de subscrições de webhooks) obrigatoriamente se integram sob `/api/v1` seguindo a mesma gramática REST e os mesmos padrões de erro.

## Evidence Found in Codebase

### Key Files
- [`src/app.ts`](file:///home/raphael/workspace/pos-ia/desafios/mba-ia-desafio-design-docs-com-ia/src/app.ts#L67-L73) - Linhas 67-73
  - Montagem do roteador da API sob `/api/v1` e registro do `errorMiddleware`.
- [`src/routes/index.ts`](file:///home/raphael/workspace/pos-ia/desafios/mba-ia-desafio-design-docs-com-ia/src/routes/index.ts#L1-L30) - Linhas 1-30
  - Composição dos sub-roteadores REST (`/auth`, `/users`, `/customers`, `/products`, `/orders`).
- [`src/shared/errors/app-error.ts`](file:///home/raphael/workspace/pos-ia/desafios/mba-ia-desafio-design-docs-com-ia/src/shared/errors/app-error.ts#L1-L17) - Linhas 1-17
  - Classe base `AppError`.
- [`src/middlewares/error.middleware.ts`](file:///home/raphael/workspace/pos-ia/desafios/mba-ia-desafio-design-docs-com-ia/src/middlewares/error.middleware.ts#L14-L65) - Linhas 14-65
  - Interceptor global e formatador JSON de erros.

### Code Evidence
```typescript
// Exemplo em src/middlewares/error.middleware.ts:14-24
export const errorMiddleware: ErrorRequestHandler = (err, req, res, _next) => {
  if (err instanceof AppError) {
    res.status(err.statusCode).json({
      error: {
        code: err.errorCode,
        message: err.message,
        ...(err.details !== undefined ? { details: err.details } : {}),
      },
    });
    return;
  }
  // ...
```

### Impact Analysis
- Introduced: 2026-06-24 (commit inicial `7ef4317`).
- Modified: Mantido consistente em todos os módulos da aplicação.
- Last change: 2026-06-24 ("init repository").
- Affects: Todos os controllers, rotas e clientes consumidores da API.
- Recent themes: "API REST", "JSON padronizado", "AppError", "tratamento centralizado de exceções".

### Alternatives (if observable)
- **GraphQL**: Permitiria queries flexíveis para os clientes, mas adicionaria complexidade de caching HTTP e curva de aprendizado desnecessária para clientes B2B tradicionais.
- **gRPC**: Ideal para comunicação interna microserviço a microserviço de altíssimo throughput, mas inadequado para parceiros B2B externos integrando via HTTP comum.

## Questions to Address in ADR (if created)

- Quais os prefixos de código de erro obrigatórios por módulo (ex: `AUTH_*`, `ORDER_*`, `WEBHOOK_*`)?
- Como documentar a especificação pública da API (ex: OpenAPI / Swagger)?

## Related Potential ADRs
- [Framework Web Express.js](./framework-web-express.md)
- [Reaproveitamento Integral dos Padrões da Codebase](../WEBHOOKS/reaproveitamento-padroes-codebase.md)

## Additional Notes
Classificado automaticamente como `must-document` pela Categoria 4 (Estilo e Protocolo de API - Step 0 da skill).
