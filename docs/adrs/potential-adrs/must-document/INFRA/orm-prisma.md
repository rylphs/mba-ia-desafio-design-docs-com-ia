# Potential ADR: ORM Prisma para Acesso a Dados e Modelagem de Esquema

**Module**: INFRA
**Category**: Data Access / Technology
**Priority**: Must Document (Score: 145)
**Date Identified**: 2026-09-13

---

## Existing ADR Context

ℹ️ **RELATED DECISIONS**

This decision relates to:
- **Banco de Dados Relacional MySQL 8.0** (`INFRA`): Dialeto e infraestrutura subjacente gerenciada pelo Prisma.
- **Padrão Transacional Outbox no MySQL** (`WEBHOOKS`): Reuso de transações interativas (`tx: Prisma.TransactionClient`) do Prisma para persistência de eventos outbox.

**When creating formal ADR**: Reference these in Related ADRs section.

---

## What Was Identified

Identificou-se a decisão de adotar o **Prisma ORM (v5.22.0)** como a camada unificada de modelagem de dados, migrações declarativas e acesso seguro ao banco de dados MySQL para a aplicação TypeScript.

O Prisma gerencia todo o esquema do banco de dados através do arquivo declarativo [`prisma/schema.prisma`](/prisma/schema.prisma), gerando automaticamente clientes tipados em tempo de compilação (`@prisma/client`). Todos os repositórios (`UserRepository`, `CustomerRepository`, `ProductRepository`, `OrderRepository`) recebem instâncias do `PrismaClient` via injeção de dependência.

Além do CRUD seguro contra injeção de SQL e geração automática de tipagens TypeScript, o Prisma provê o mecanismo de **Transações Interativas** (`prisma.$transaction(async (tx) => { ... })`), que permite encapsular múltiplas consultas, verificações condicionais de regras de negócio (como checagem de estoque e máquina de estados) e operações de escrita dentro de uma única transação atômica do MySQL.

## Why This Might Deserve an ADR

- **Impact**: Fundacional para a camada de persistência. A tipagem estrita gerada pelo Prisma permeia serviços, repositórios, schemas Zod e controllers em toda a aplicação.
- **Trade-offs**: O Prisma provê DX (*Developer Experience*) excepcional e segurança de tipos em tempo de compilação, mas gera uma camada de abstração com query engine binária e menor flexibilidade para consultas altamente analíticas ou comandos customizados de banco (como `SKIP LOCKED`), exigindo `$queryRaw` quando necessário.
- **Complexity**: Média. Simplifica a manutenção das migrações (`prisma migrate dev`), mas requer disciplina na instanciação do `PrismaClient` (evitar múltiplas instâncias no mesmo processo para não estourar pools de conexão).
- **Team Knowledge**: Essencial. Todos os desenvolvedores interagem com a API gerada do Prisma para implementar regras e repositórios.
- **Future Implications**: Novas entidades (como `WebhookSubscription`, `WebhookOutbox`, `WebhookDelivery` e `WebhookDeadLetter`) serão adicionadas ao `schema.prisma` e versionadas via migrações Prisma.

## Evidence Found in Codebase

### Key Files
- [`prisma/schema.prisma`](/prisma/schema.prisma#L1-L139) - Linhas 1-139
  - Declaração de models (`User`, `Customer`, `Product`, `Order`, `OrderItem`, `OrderStatusHistory`, `OrderNumberSequence`).
- [`package.json`](/package.json#L26-L48) - Linhas 26 e 48
  - `@prisma/client: 5.22.0` e `prisma: 5.22.0`.
- [`src/config/database.ts`](/src/config/database.ts#L1-L10) - Linhas 1-10
  - Instanciação centralizada singleton do `PrismaClient`.
- [`src/modules/orders/order.service.ts`](/src/modules/orders/order.service.ts#L24-L179) - Linhas 24-179
  - Uso de `Prisma.TransactionClient` e `prisma.$transaction`.

### Code Evidence
```typescript
// Exemplo em src/modules/orders/order.service.ts:24-30
type TxClient = Prisma.TransactionClient;

export class OrderService {
  constructor(
    private readonly orders: OrderRepository,
    private readonly prisma: PrismaClient,
  ) {}
  // ...
```

### Impact Analysis
- Introduced: 2026-06-24 (commit inicial `7ef4317`).
- Modified: Modelo de dados estável com migração inicial em `prisma/migrations/20260519182739_init`.
- Last change: 2026-06-24 ("init repository").
- Affects: Todos os repositories, services, schemas de dados e migrações.
- Recent themes: "Prisma ORM", "segurança de tipos", "migrações", "transações interativas".

### Alternatives (if observable)
- **TypeORM / Sequelize**: ORMs tradicionais baseados em classes e decorators, descartados pela maior fragilidade de tipagem e complexidade de decorators no ESM moderno.
- **Kysely / SQL Puro**: Construtor de queries SQL fortemente tipado, descartado em favor do ecossistema de migrações declarativas completas do Prisma.

## Questions to Address in ADR (if created)

- Como configurar os limites do connection pool do Prisma para coexistir de forma equilibrada entre o processo do servidor HTTP e o processo do worker?
- Como será estruturada a próxima migração para criação das tabelas do subsistema de webhooks?

## Related Potential ADRs
- [Banco de Dados Relacional MySQL 8.0](./banco-de-dados-relacional-mysql.md)
- [Padrão Transacional Outbox no MySQL](../../done/WEBHOOKS/padrao-transacional-outbox-no-mysql.md)

## Additional Notes
Classificado automaticamente como `must-document` pela Categoria 3 (ORM/Data Access Layer - Step 0 da skill).
