# Potential ADR: Autenticação Stateless com JWT e Controle de Acesso Baseado em Papéis (RBAC)

**Module**: AUTH
**Category**: Security
**Priority**: Must Document (Score: 125)
**Date Identified**: 2026-09-13

---

## Existing ADR Context

ℹ️ **RELATED DECISIONS**

This decision relates to:
- **Framework Web Express.js** (`INFRA`): Onde os middlewares `authenticate` e `requireRole` interceptam requisições.
- **Autenticação e Integridade via HMAC-SHA256** (`WEBHOOKS`): Complementa a segurança de usuários da plataforma com a segurança de clientes externos.
- **Política de Retry com Backoff Exponencial e Tabela DLQ Dedicada** (`WEBHOOKS`): Utiliza `requireRole('ADMIN')` para restringir a ação de replay de DLQ.

**When creating formal ADR**: Reference these in Related ADRs section.

---

## What Was Identified

Identificou-se a decisão de arquitetura de segurança que adota tokens **JSON Web Tokens (JWT)** assinados de forma *stateless* e um mecanismo declarativo de **Controle de Acesso Baseado em Papéis (RBAC - Role-Based Access Control)** para autenticação e autorização de usuários e operadores da API REST.

As credenciais dos usuários são validadas contra hashes seguros gerados com `bcrypt` (salting rounds padrão) em [`AuthService.login`](/src/modules/auth/auth.service.ts#L22-L42). Em caso de sucesso, o serviço emite um token JWT contendo os claims essenciais no payload: `sub` (User ID), `email` e `role` (`ADMIN` ou `OPERATOR`), assinado com uma chave secreta simétrica (`JWT_SECRET`) e tempo de expiração (`JWT_EXPIRES_IN`).

A proteção dos endpoints é realizada por dois middlewares reutilizáveis em [`src/middlewares/auth.middleware.ts`](/src/middlewares/auth.middleware.ts):
1. `authenticate`: Valida o cabeçalho `Authorization: Bearer <token>`, decodifica o payload e injeta os dados do usuário em `req.user`.
2. `requireRole(...roles)`: Valida se o usuário autenticado possui o papel requerido, rejeitando a requisição com `ForbiddenError (403)` em caso negativo.

## Why This Might Deserve an ADR

- **Impact**: Define todo o modelo de segurança e governança de acesso aos recursos de pedidos, clientes, usuários e operações administrativas do sistema.
- **Trade-offs**: A abordagem stateless com JWT permite escalabilidade horizontal sem necessidade de compartilhar sessões em banco de dados ou Redis, mas inviabiliza revogação instantânea de tokens emitidos antes do seu vencimento natural sem a implementação de uma *blacklist*.
- **Complexity**: Média. Bem encapsulada na camada de middlewares do Express.
- **Team Knowledge**: Essencial. Todo desenvolvedor precisa saber como proteger novos endpoints e exigir papéis específicos (ex: `requireRole('ADMIN')` para replay da DLQ de webhooks).
- **Future Implications**: Estabelece que futuras integrações de API administrativa utilizem os mesmos papéis e tokens do sistema.

## Evidence Found in Codebase

### Key Files
- [`src/middlewares/auth.middleware.ts`](/src/middlewares/auth.middleware.ts#L1-L62) - Linhas 1-62
  - Middlewares `authenticate` e `requireRole`.
- [`src/modules/auth/auth.service.ts`](/src/modules/auth/auth.service.ts#L22-L42) - Linhas 22-42
  - Validação com `bcrypt` e geração de token JWT.
- [`prisma/schema.prisma`](/prisma/schema.prisma#L11-L14) - Linhas 11-14 e 25-38
  - Enum `UserRole { ADMIN, OPERATOR }` e modelo `User`.

### Code Evidence
```typescript
// src/middlewares/auth.middleware.ts:49-61
export function requireRole(...roles: AuthUser['role'][]): RequestHandler {
  return (req, _res, next) => {
    if (!req.user) {
      next(new UnauthorizedError());
      return;
    }
    if (!roles.includes(req.user.role)) {
      next(new ForbiddenError('Insufficient permissions'));
      return;
    }
    next();
  };
}
```

### Impact Analysis
- Introduced: 2026-06-24 (commit inicial `7ef4317`).
- Modified: Modelo de autorização consolidado.
- Last change: 2026-06-24 ("init repository").
- Affects: Rotas de autenticação, usuários, pedidos e novos endpoints de webhooks (como replay de DLQ).
- Recent themes: "JWT", "RBAC", "ADMIN vs OPERATOR", "stateless auth".

### Alternatives (if observable)
- **Sessões Stateful com Cookies e Redis**: Descartadas pela necessidade de manter uma camada de infraestrutura de cache compartilhada apenas para gerenciamento de sessões.
- **OAuth2 / OIDC Externo (Auth0, Keycloak)**: Descartado pela simplicidade e autonomia de ter a autenticação embutida no próprio OMS para usuários internos.

## Questions to Address in ADR (if created)

- Qual o tempo ideal de expiração dos tokens JWT para equilibrar segurança e conveniência do operador?
- Como estender o RBAC se novos papéis forem requeridos (ex: `AUDITOR`, `INTEGRATION_CLIENT`)?

## Related Potential ADRs
- [Framework Web Express.js](../INFRA/framework-web-express.md)
- [Autenticação e Integridade via HMAC-SHA256 com Secret por Endpoint](../../done/WEBHOOKS/autenticacao-integridade-hmac-sha256-secret-por-endpoint.md)

## Additional Notes
Classificado como `must-document` por ser infraestrutura crítica de segurança e autenticação voltada a usuários (Step 0 - Infraestrutura Específica de Domínio).
