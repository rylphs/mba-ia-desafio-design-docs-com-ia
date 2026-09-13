# Codebase Architecture Mapping

## Project Overview

- **Project Name**: `order-management-api` (Order Management System REST API)
- **Purpose**: Sistema de gerenciamento de pedidos (Order Management System - OMS) com suporte a cadastro e gestão de usuários, clientes, catálogo de produtos, controle transacional de estoque e ciclo de vida de pedidos com máquina de estados e histórico de auditoria. Adicionalmente, especificação do novo subsistema de Webhooks de Notificação de Pedidos para integração B2B em tempo real.
- **Type**: Backend RESTful API & Asynchronous Event Processing Service
- **Languages**: TypeScript (ESM, Target ES2022, Node.js >=20)
- **Primary Framework**: Express 4.21.1

## Technology Stack

- **Runtime & Language**: Node.js (>=20), TypeScript (v5.6.3), Execution via `tsx` (v4.19.2)
- **Web Framework**: Express.js (v4.21.1)
- **Data Access & ORM**: Prisma ORM (v5.22.0) com `@prisma/client`
- **Database**: MySQL 8.0 (executando via Docker Compose com charset `utf8mb4_unicode_ci`)
- **Authentication & Security**: JWT (`jsonwebtoken` v9.0.2), Password Hashing com `bcrypt` (v5.1.1), Payload Signing via Node.js Native Crypto (`HMAC-SHA256`)
- **Validation**: Zod (v3.23.8)
- **Logging**: Pino (v9.5.0), `pino-http` (v10.3.0), `pino-pretty` (v11.3.0)
- **Testing**: Vitest (v2.1.4), Supertest (v7.0.0)
- **Infrastructure / Containerization**: Docker Compose (`mysql:8.0`)

## Context Notes

**Source Files**:
- `TRANSCRICAO.md`: Transcrição da reunião técnica sobre o Sistema de Webhooks de Notificação de Pedidos realizada entre Larissa (Tech Lead), Marcos (PM), Bruno (Eng. Pedidos), Diego (Eng. Plataforma) e Sofia (Eng. Segurança).
- `docs/adrs/must-include.md`: Registro explícito das principais decisões técnicas obrigatórias levantadas na reunião.
- `README.md`: Especificação e diretrizes do desafio de engenharia de software com IA.

**Key Insights**:
- **Architectural patterns mentioned**:
  - Padrão Transacional Outbox (*Transactional Outbox Pattern*) para publicação confiável e atômica de eventos no MySQL.
  - Processamento Assíncrono com Polling Worker desacoplado em processo separado (`src/worker.ts`) com intervalo de 2 segundos.
  - Semântica de entrega *At-Least-Once* com desduplicação por identificador único de evento (`X-Event-Id`).
  - Resiliência com Política de Retry Exponencial (5 tentativas: 1m, 5m, 30m, 2h, 12h) e Dead Letter Queue (DLQ) persistida em tabela dedicada com replay manual via endpoint admin (`POST /admin/webhooks/dead-letter/:id/replay`).
  - Assinatura criptográfica *HMAC-SHA256* com segredo exclusivo por endpoint e política de rotação com *grace period* de 24h.
  - Máquina de Estados Finita e controle transacional ACID de estoque dentro do método `OrderService.changeStatus`.
- **Business domains identified**:
  - Gestão de Acesso e Identidade (Auth & Users)
  - Gestão de Clientes B2B (Customers)
  - Catálogo de Produtos e Inventário de Estoque (Products)
  - Vendas, Pedidos e Mudanças de Status com Auditoria (Orders)
  - Notificações de Eventos em Tempo Real para Clientes Externos (Webhooks)
- **Module boundaries documented**:
  - Organização modular estrita sob `src/modules/*` dividida em camadas horizontais homogêneas: `*.controller.ts`, `*.service.ts`, `*.repository.ts`, `*.routes.ts`, `*.schemas.ts`.
  - Tratamento centralizado de erros via `AppError` e middleware Express em `src/middlewares/error.middleware.ts`.
  - Controle de autorização baseado em perfis (RBAC: `ADMIN` e `OPERATOR`) via `requireRole` em `src/middlewares/auth.middleware.ts`.
- **Technologies documented**:
  - Alinhamento total entre o código descoberto (Node.js, Express, Prisma, MySQL, Pino, Zod) e as discussões arquiteturais documentadas na transcrição.
- **Discrepancies / Gaps Identified**:
  - O código base atual cobre os módulos `auth`, `users`, `customers`, `products`, `orders` e `infra`, mas não possui ainda o módulo `src/modules/webhooks` nem `src/worker.ts`, cujo design e especificação arquitetural são o foco deste pacote de decisões.

## System Modules

### Module Index
1. `AUTH` - Autenticação e Emissão de Tokens: Gerenciamento de credenciais e tokens JWT.
2. `USERS` - Gestão de Usuários e Perfis: Cadastro de operadores e administradores com hashing seguro.
3. `CUSTOMERS` - Gestão de Clientes: Cadastro e consulta de clientes B2B e endereços.
4. `PRODUCTS` - Catálogo e Estoque: Controle de produtos, preços em centavos e saldo de estoque.
5. `ORDERS` - Pedidos e Ciclo de Vida: Transações ACID, máquina de estados e auditoria de status.
6. `INFRA` - Infraestrutura, Persistência e Transversais: Docker, Prisma, MySQL, middlewares e log.
7. `WEBHOOKS` - Sistema de Webhooks e Notificações: Outbox, polling worker, retry, DLQ e HMAC.

---

### [AUTH]: Autenticação e Emissão de Tokens
**Purpose**: Responsável pela autenticação de usuários, validação de senhas via bcrypt, emissão e assinatura de tokens JWT e verificação de sessão.
**Location**: `src/modules/auth/*`
**Key Components**: `AuthController`, `AuthService`, `AuthSchemas`, `AuthRoutes`
**Technologies**: `jsonwebtoken`, `bcrypt`, `zod`, `express`
**Dependencies**: Módulo `USERS` (`UserRepository`, `UserService`), `src/config/env.ts`
**Patterns**: Layered Architecture, Token-Based Authentication (JWT Bearer)
**Key Files**:
- `src/modules/auth/auth.service.ts`
- `src/modules/auth/auth.controller.ts`
- `src/modules/auth/auth.routes.ts`
- `src/modules/auth/auth.schemas.ts`
**Scope**: Small - 4 arquivos

---

### [USERS]: Gestão de Usuários e Perfis
**Purpose**: Gerencia o ciclo de vida dos operadores e administradores do sistema, atribuição de papéis (`ADMIN`, `OPERATOR`) e hashing de senhas.
**Location**: `src/modules/users/*`
**Key Components**: `UserController`, `UserService`, `UserRepository`, `UserSchemas`, `UserRoutes`
**Technologies**: Prisma Client, `bcrypt`, `zod`, `express`
**Dependencies**: `PrismaClient`, `src/shared/errors/*`
**Patterns**: Layered Architecture (Controller-Service-Repository), RBAC
**Key Files**:
- `src/modules/users/user.service.ts`
- `src/modules/users/user.repository.ts`
- `src/modules/users/user.controller.ts`
**Scope**: Small - 5 arquivos

---

### [CUSTOMERS]: Gestão de Clientes
**Purpose**: Manutenção do cadastro de clientes B2B da plataforma, validação de documentos cadastrais e armazenamento flexível de endereços estruturados em JSON.
**Location**: `src/modules/customers/*`
**Key Components**: `CustomerController`, `CustomerService`, `CustomerRepository`, `CustomerSchemas`, `CustomerRoutes`
**Technologies**: Prisma Client, `zod`, `express`
**Dependencies**: `PrismaClient`, `src/shared/errors/*`
**Patterns**: Layered Architecture, Document/JSON Storage in Relational DB
**Key Files**:
- `src/modules/customers/customer.service.ts`
- `src/modules/customers/customer.repository.ts`
- `src/modules/customers/customer.controller.ts`
**Scope**: Small - 5 arquivos

---

### [PRODUCTS]: Catálogo e Estoque
**Purpose**: Gestão do catálogo de produtos, controle de unicidade de SKUs, precisão contábil com preços em centavos e rastreamento atômico de quantidade em estoque.
**Location**: `src/modules/products/*`
**Key Components**: `ProductController`, `ProductService`, `ProductRepository`, `ProductSchemas`, `ProductRoutes`
**Technologies**: Prisma Client, `zod`, `express`
**Dependencies**: `PrismaClient`, `src/shared/errors/*`
**Patterns**: Layered Architecture, Integer Representation of Currency
**Key Files**:
- `src/modules/products/product.service.ts`
- `src/modules/products/product.repository.ts`
- `src/modules/products/product.controller.ts`
**Scope**: Small - 5 arquivos

---

### [ORDERS]: Pedidos e Ciclo de Vida
**Purpose**: Criação de pedidos, reserva de número sequencial, validação e execução de máquina de estados de status, débito/estorno atômico de estoque e auditoria histórica de alterações de status sob transação do banco de dados.
**Location**: `src/modules/orders/*`
**Key Components**: `OrderController`, `OrderService`, `OrderRepository`, `OrderSchemas`, `OrderRoutes`, `order.status.ts`
**Technologies**: Prisma Interactive Transactions (`$transaction`), `zod`, `express`
**Dependencies**: `PrismaClient`, `CUSTOMERS`, `PRODUCTS`, `USERS`, `src/shared/errors/*`
**Patterns**: State Machine Pattern, Unit of Work / Transaction Script, Atomic Inventory Locking
**Key Files**:
- `src/modules/orders/order.service.ts`
- `src/modules/orders/order.status.ts`
- `src/modules/orders/order.repository.ts`
- `src/modules/orders/order.controller.ts`
**Scope**: Medium - 6 arquivos

---

### [INFRA]: Infraestrutura, Persistência e Transversais
**Purpose**: Prover serviços de inicialização de servidor, pooling e conexão com banco de dados MySQL, definições de esquema Prisma, middlewares de autorização, tratamento global de exceções e logger estruturado com rastreabilidade de requisições.
**Location**: `src/config/*`, `src/middlewares/*`, `src/shared/*`, `prisma/*`, `docker-compose.yml`
**Key Components**: `buildApp` (`src/app.ts`), `server.ts`, `prisma/schema.prisma`, `auth.middleware.ts`, `error.middleware.ts`, `requestLogger`, `AppError`
**Technologies**: Express 4, MySQL 8, Prisma 5, Pino, Docker Compose
**Dependencies**: Driver MySQL, Node.js runtime
**Patterns**: Centralized Error Handler, Middleware Pipeline, Environment Configuration with Schema, Structured JSON Logging
**Key Files**:
- `src/app.ts`
- `src/server.ts`
- `src/config/database.ts`
- `src/config/env.ts`
- `src/middlewares/auth.middleware.ts`
- `src/middlewares/error.middleware.ts`
- `src/shared/errors/app-error.ts`
- `prisma/schema.prisma`
- `docker-compose.yml`
**Scope**: Medium - ~15 arquivos

---

### [WEBHOOKS]: Sistema de Webhooks e Notificações (Feature em Especificação)
**Purpose**: Prover entrega assíncrona, confiável e resiliente de notificações de eventos de mudança de status de pedidos para clientes B2B. Inclui persistência atômica via Outbox no MySQL, worker desacoplado com polling, garantia *at-least-once*, política de retentativas com backoff exponencial, Dead Letter Queue (DLQ) persistida para auditoria/replay manual e autenticação criptográfica via HMAC-SHA256 com secret rotacionável por endpoint.
**Location**: `src/modules/webhooks/*` (design da feature), `src/worker.ts` (entrypoint), integrado em `src/modules/orders/order.service.ts`
**Key Components**: `WebhookService`, `WebhookRepository`, `WebhookWorker`, `WebhookSigner`, `DLQReplayController`
**Technologies**: Node.js, Prisma Client, MySQL 8, Crypto (`crypto.createHmac`), Pino, Zod
**Dependencies**: `ORDERS`, `INFRA` (MySQL, Prisma, Logger, Error Handler, Middlewares)
**Patterns**: Transactional Outbox Pattern, Polling Consumer, At-Least-Once Delivery, Exponential Backoff with DLQ, Cryptographic Payload Signing (HMAC-SHA256)
**Key Files**:
- `src/modules/orders/order.service.ts` (Ponto de integração transacional)
- `src/worker.ts` (Processo do worker)
- `prisma/schema.prisma` (Tabelas `webhook_outbox`, `webhook_subscriptions`, `webhook_deliveries`, `webhook_dead_letter`)
**Scope**: Medium

---

## Cross-Cutting Concerns

- **Infrastructure**: Instância MySQL 8.0 rodando em container Docker com volumes persistentes (`oms_mysql_data`) e collation `utf8mb4_unicode_ci`.
- **Auth & Security**: Autenticação stateless via JWT Bearer Tokens, controle de permissões por roles (`ADMIN`, `OPERATOR`) e assinatura de mensagens externas via HMAC-SHA256 com chaves segregadas por cliente/endpoint.
- **Data Layer**: Acesso fortemente tipado com Prisma Client, migrações automatizadas via Prisma Migrate, transações interativas atômicas (`$transaction`), identificadores primários UUID v4 char(36) e valores monetários armazenados como inteiros em centavos (`*Cents`).
- **API Layer**: API RESTful sob prefixo `/api/v1`, validação estrita de payload de entrada com esquemas Zod, serialização de paginação consistente e respostas HTTP padronizadas.
- **Integrations**: Comunicação outbound baseada em Webhooks HTTP POST com timeouts estritos de 10s, limites de payload de 64KB, idempotência delegada via cabeçalho `X-Event-Id` e rastreabilidade temporal com `X-Timestamp`.
