# Potential ADR: Estratégia de Identificadores UUID v4 como Chave Primária

**Module**: INFRA
**Category**: Architecture / Data Modeling
**Priority**: Consider (Score: 75)
**Date Identified**: 2026-09-13

---

## Existing ADR Context

ℹ️ **RELATED DECISIONS**

This decision relates to:
- **Banco de Dados Relacional MySQL 8.0** (`INFRA`): Onde os campos de chave primária são persistidos como colunas `CHAR(36)`.
- **ORM Prisma** (`INFRA`): Diretiva `@default(uuid())` que instrui a geração de identificadores universais pelo Prisma.
- **Padrão Transacional Outbox no MySQL** (`WEBHOOKS`): Reafirmação expressa na reunião por Larissa e Diego de que novos modelos da outbox devem seguir o padrão de UUIDs da aplicação.

**When creating formal ADR**: Reference these in Related ADRs section.

---

## What Was Identified

Identificou-se a decisão arquitetural transversal de adotar identificadores universais **UUID v4 (Universally Unique Identifier)** como a estratégia padrão de chave primária (`@id`) em praticamente todas as entidades do banco de dados relacional (`User`, `Customer`, `Product`, `Order`, `OrderItem`, `OrderStatusHistory`), armazenados em colunas MySQL com tipo estrito `CHAR(36)`.

A única exceção deliberada à regra de UUID é a tabela de controle de numeração legível `OrderNumberSequence`, que utiliza um `Int` autoincremental para gerar números sequenciais humanizados de pedidos (ex: `ORD-000001`).

Essa estratégia foi expressamente ratificada na reunião técnica do sistema de webhooks (registrada em `TRANSCRICAO.md` linhas 302-306), onde Diego e Larissa confirmaram que a nova modelagem de tabelas de outbox, subscrições e DLQ deve manter a coerência sistêmica e utilizar identificadores UUID em detrimento de IDs autoincrementais inteiros.

## Why This Might Deserve an ADR

- **Impact**: Afeta todas as tabelas do banco, relacionamentos de chave estrangeira, índices primários e os contratos públicos de todos os endpoints da API REST e headers de webhooks.
- **Trade-offs**: UUIDs evitam ataques de enumeração direta de recursos na API (IDOR) e permitem a geração descentralizada e concorrente de identificadores sem necessidade de consultar o banco antes de criar transações. Por outro lado, ocupam 36 bytes por registro (contra 4 ou 8 bytes de inteiros/bigint) e provocam fragmentação de índices B-Tree no MySQL (devido à aleatoriedade do UUID v4), impactando ligeiramente a eficiência de I/O em grandes volumes.
- **Complexity**: Baixa na geração, mas com impacto duradouro em performance e migração.
- **Team Knowledge**: Alto. Engenheiros precisam saber que nenhuma tabela de domínio deve usar `AUTO_INCREMENT` para sua chave primária.
- **Future Implications**: Torna inviável migrar de forma simples para chaves autoincrementais futuramente devido a relacionamentos estabelecidos em produção.

## Evidence Found in Codebase

### Key Files
- [`prisma/schema.prisma`](/prisma/schema.prisma#L25-L131) - Linhas 25-131
  - Modelos `User`, `Customer`, `Product`, `Order`, `OrderItem`, `OrderStatusHistory` configurados com `@default(uuid()) @db.Char(36)`.
- [`TRANSCRICAO.md`](/TRANSCRICAO.md#L302-L306) - Linhas 302-306
  - Alinhamento explícito entre Diego e Larissa definindo UUID para a outbox para manter conformidade com o padrão do projeto.
- [`package.json`](/package.json#L32) - Linha 32
  - Biblioteca `uuid` v11.0.3 instalada nas dependências de produção.

### Code Evidence
```prisma
// Exemplo em prisma/schema.prisma:25-26 e 74-75
model User {
  id String @id @default(uuid()) @db.Char(36)
  // ...
}

model Order {
  id String @id @default(uuid()) @db.Char(36)
  // ...
}
```

```text
// Diálogo em TRANSCRICAO.md:302-306
[09:51] Diego: Quando a gente for modelar a outbox, prefere id auto incremental ou UUID?
[09:51] Larissa: UUID, segue o padrão do resto do projeto. Tudo é uuid.
[09:51] Diego: Beleza, só queria confirmar.
```

### Impact Analysis
- Introduced: 2026-06-24 (commit inicial `7ef4317`) e ratificado para Webhooks em 2026-09-13.
- Modified: Padronizado em todos os modelos de domínio.
- Last change: 2026-09-13.
- Affects: Todas as tabelas, chaves estrangeiras, índices e schemas de validação Zod.
- Recent themes: "UUID v4", "CHAR(36)", "chave primária descentralizada", "anti-enumeração".

### Alternatives (if observable)
- **Integers / Bigint Auto-increment**: Descartados para chaves primárias de domínio para prevenir adivinhação de recursos e garantir uniformidade em integrações externas.
- **UUID v7 (Time-ordered / Sequential)**: Alternativa moderna que minimizaria a fragmentação de B-Tree no MySQL, preservando os benefícios de unicidade global, que pode ser considerada em evoluções futuras da engine.

## Questions to Address in ADR (if created)

- O projeto deve considerar a adoção de UUID v7 para tabelas com volume massivo de inserções (como histórico e outbox) para reduzir fragmentação de índice?
- Qual a estratégia de validação no Zod (uso sistemático de `z.string().uuid()`)?

## Related Potential ADRs
- [Banco de Dados Relacional MySQL 8.0](../../must-document/INFRA/banco-de-dados-relacional-mysql.md)
- [Padrão Transacional Outbox no MySQL](../../done/WEBHOOKS/padrao-transacional-outbox-no-mysql.md)

## Additional Notes
Avaliando os 3 E's (Estrutural, Evidente, Estável), a decisão obteve pontuação 75/150 (Escopo: 25, Custo de Mudança: 25, Conhecimento da Equipe: 25), sendo posicionada na faixa de prioridade média (`consider/`).
