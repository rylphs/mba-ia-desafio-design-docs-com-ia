# Potential ADR: Framework Web Express.js para Construção da API REST

**Module**: INFRA
**Category**: Platform
**Priority**: Must Document (Score: 140)
**Date Identified**: 2026-09-13

---

## Existing ADR Context

ℹ️ **RELATED DECISIONS**

This decision relates to:
- **Arquitetura de API REST com Tratamento Centralizado de Erros** (`INFRA`): Estilo de design da API viabilizado pelo pipeline de middlewares do Express.
- **Worker Desacoplado via Polling** (`WEBHOOKS`): O processo worker permanece desacoplado do ciclo de vida da aplicação Express.
- **Autenticação Stateless com JWT e RBAC** (`AUTH`): Middlewares de autenticação e autorização construídos para a cadeia de handlers do Express.

**When creating formal ADR**: Reference these in Related ADRs section.

---

## What Was Identified

Identificou-se a decisão de adotar o **Express.js (v4.21.1)** como o framework web HTTP primário para a construção da API REST do Order Management System (OMS).

A aplicação utiliza a arquitetura funcional baseada em *application builder* (`buildApp`) e injeção de dependências modular manual através de `buildControllers(prisma)` em [`src/app.ts`](file:///home/raphael/workspace/pos-ia/desafios/mba-ia-desafio-design-docs-com-ia/src/app.ts#L26-L76). Isso viabiliza testes de integração isolados com Supertest sem inicializar portas de rede reais.

O pipeline de middlewares do Express é amplamente aproveitado para:
- Desativação do cabeçalho invasivo de tecnologia (`app.disable('x-powered-by')`).
- Parsing automático de corpos JSON com teto de 1MB (`express.json({ limit: '1mb' })`).
- Logging estruturado assíncrono de cada requisição via `pino-http`.
- Roteamento modular desacoplado sob `/api/v1`.
- Rota fallback para 404 (`NotFoundError`).
- Tratamento centralizado e unificado de exceções via `errorMiddleware` de 4 parâmetros.

## Why This Might Deserve an ADR

- **Impact**: Estrutural. Define todo o modelo de concorrência, o formato de tratamento de requisições, a interoperabilidade de middlewares e a arquitetura de testes HTTP da aplicação.
- **Trade-offs**: O Express 4 é maduro, estável, tem o maior ecossistema de bibliotecas do Node.js e quase zero atrito de adoção, mas não possui tipagem TypeScript nativa de ponta a ponta para validação de esquemas (suprida pelo Zod) e possui menor throughput bruto se comparado a frameworks como Fastify.
- **Complexity**: Baixa na manutenção e alta facilidade de composição.
- **Team Knowledge**: Universal. Praticamente qualquer engenheiro Node.js / TypeScript domina o modelo de rotas e middlewares do Express.
- **Future Implications**: Determina que novos módulos (como Webhooks) integrem seus endpoints via `Router` padrão do Express.

## Evidence Found in Codebase

### Key Files
- [`package.json`](file:///home/raphael/workspace/pos-ia/desafios/mba-ia-desafio-design-docs-com-ia/package.json#L28) - Linha 28
  - Dependência principal `"express": "4.21.1"`.
- [`src/app.ts`](file:///home/raphael/workspace/pos-ia/desafios/mba-ia-desafio-design-docs-com-ia/src/app.ts#L1-L76) - Linhas 1-76
  - Funções `buildControllers` e `buildApp`.
- [`src/server.ts`](file:///home/raphael/workspace/pos-ia/desafios/mba-ia-desafio-design-docs-com-ia/src/server.ts#L1-L28) - Linhas 1-28
  - Bootstrap do servidor ouvindo na porta configurada com encerramento gracioso.

### Code Evidence
```typescript
// src/app.ts:55-75
export function buildApp(deps: AppDependencies): Express {
  const app = express();

  app.disable('x-powered-by');
  app.use(express.json({ limit: '1mb' }));
  app.use(requestLogger);

  app.get('/health', (_req, res) => {
    res.status(200).json({ status: 'ok' });
  });

  const controllers = buildControllers(deps.prisma);
  app.use('/api/v1', buildApiRouter(controllers));

  app.use((req, _res, next) => {
    next(new NotFoundError(`Route ${req.method} ${req.originalUrl}`));
  });

  app.use(errorMiddleware);

  return app;
}
```

### Impact Analysis
- Introduced: 2026-06-24 (commit inicial `7ef4317`).
- Modified: Base estável do backend da aplicação.
- Last change: 2026-06-24 ("init repository").
- Affects: Todos os controllers, rotas, middlewares e testes da suíte `tests/`.
- Recent themes: "Express 4", "pipeline de middleware", "buildApp", "REST API".

### Alternatives (if observable)
- **Fastify**: Ofereceria maior performance de serialização e suporte nativo a esquemas JSON, mas a simplicidade e estabilidade do ecossistema Express foi priorizada.
- **NestJS**: Framework mais opinado com injeção de dependência via decorators, mas considerado desnecessariamente verboso e complexo para os objetivos da aplicação.

## Questions to Address in ADR (if created)

- O projeto planeja migração para o Express 5 no futuro para suporte nativo a async/await em middlewares sem necessidade de tratamento manual?
- Quais as boas práticas de modularização de rotas com o router do Express mantidas no projeto?

## Related Potential ADRs
- [Arquitetura de API REST com Tratamento Centralizado de Erros](./arquitetura-api-rest.md)
- [Reaproveitamento Integral dos Padrões da Codebase](../WEBHOOKS/reaproveitamento-padroes-codebase.md)

## Additional Notes
Classificado automaticamente como `must-document` pela Categoria 2 (Framework/Plataforma Primária - Step 0 da skill).
