# Potential ADR: Banco de Dados Relacional MySQL 8.0 como Armazenamento Primário

**Module**: INFRA
**Category**: Technology
**Priority**: Must Document (Score: 145)
**Date Identified**: 2026-09-13

---

## Existing ADR Context

ℹ️ **RELATED DECISIONS**

This decision relates to:
- **ORM Prisma** (`INFRA`): Camada de mapeamento objeto-relacional e migrações que abstrai o acesso ao MySQL.
- **Padrão Transacional Outbox no MySQL** (`WEBHOOKS`): Mecanismo que utiliza a capacidade transacional ACID do MySQL para publicação de eventos.
- **Estratégia de Identificadores UUID v4** (`INFRA`): Estratégia de tipos de dados de chave primária (`CHAR(36)`) no MySQL.

**When creating formal ADR**: Reference these in Related ADRs section.

---

## What Was Identified

Identificou-se a decisão de adotar o sistema gerenciador de banco de dados relacional **MySQL 8.0** como o motor primário e unificado de armazenamento de dados da aplicação. O banco de dados é executado em container Docker padronizado via `docker-compose.yml`, configurado com o charset `utf8mb4` e collation `utf8mb4_unicode_ci`, com volume persistente local (`oms_mysql_data`).

A persistência do sistema armazena todos os agregados do domínio de negócio em tabelas relacionais com chaves estrangeiras estritas, constraints de unicidade (e-mails, SKUs, números de pedidos) e suporte a colunas do tipo JSON para informações de endereços e payloads de auditoria.

O suporte a transações ACID com isolamento rigoroso no MySQL é peça central para operações de missão crítica do OMS, em particular a atomicidade do método `OrderService.changeStatus`, que orquestra a atualização de status de pedidos, histórico de auditoria, reservas e devoluções de estoque de produtos, e o futuro registro atômico na tabela de outbox.

## Why This Might Deserve an ADR

- **Impact**: Fundacional. Todo o modelo de dados, integridade referencial, garantias transacionais ACID e capacidade de concorrência do OMS dependem da infraestrutura do MySQL.
- **Trade-offs**: O MySQL 8.0 oferece robustez, consistência forte e ampla maturidade de mercado, mas não possui mecanismos reativos nativos de notificação em rede para processos externos (como o `LISTEN/NOTIFY` do PostgreSQL), exigindo a técnica de polling para leitura de tabelas de outbox.
- **Complexity**: Alta para migrações estruturais e mudanças de engine, mas bem isolada pela utilização do Prisma ORM.
- **Team Knowledge**: Essencial. Todos os engenheiros precisam compreender o modelo relacional, transações e índices existentes.
- **Future Implications**: Qualquer decisão futura de escalabilidade (leitura/escrita, réplicas ou sharding) parte desta escolha de banco.

## Evidence Found in Codebase

### Key Files
- [`docker-compose.yml`](file:///home/raphael/workspace/pos-ia/desafios/mba-ia-desafio-design-docs-com-ia/docker-compose.yml#L1-L29) - Linhas 1-29
  - Declaração do serviço `mysql:8.0`, portas, volumes e flags de inicialização (`utf8mb4`).
- [`prisma/schema.prisma`](file:///home/raphael/workspace/pos-ia/desafios/mba-ia-desafio-design-docs-com-ia/prisma/schema.prisma#L5-L9) - Linhas 5-9
  - Configuração do datasource com `provider = "mysql"`.
- [`src/config/database.ts`](file:///home/raphael/workspace/pos-ia/desafios/mba-ia-desafio-design-docs-com-ia/src/config/database.ts#L1-L10) - Linhas 1-10
  - Inicialização do client de banco de dados.

### Code Evidence
```yaml
# docker-compose.yml:1-25
services:
  mysql:
    image: mysql:8.0
    container_name: oms-mysql
    restart: unless-stopped
    ports:
      - '3306:3306'
    volumes:
      - oms_mysql_data:/var/lib/mysql
    command:
      - --default-authentication-plugin=mysql_native_password
      - --character-set-server=utf8mb4
      - --collation-server=utf8mb4_unicode_ci
```

### Impact Analysis
- Introduced: 2026-06-24 (commit inicial `7ef4317`).
- Modified: Mantido estável desde o bootstrap do projeto.
- Last change: 2026-06-24 ("init repository").
- Affects: Todas as tabelas, transações, migrações e módulos da aplicação.
- Recent themes: "infraestrutura", "persistência relacional", "ACID", "utf8mb4".

### Alternatives (if observable)
- **PostgreSQL**: Ofereceria `NOTIFY/LISTEN` nativo, mas a decisão do projeto foi consolidar a infraestrutura em MySQL 8.0, amplamente disponível e operacionalmente dominada pela equipe.
- **NoSQL (ex: MongoDB)**: Descartado pelas fortes restrições transacionais relacionais requeridas pelo ciclo de vida de pedidos e controle de estoque concorrente.

## Questions to Address in ADR (if created)

- Quais as estratégias de backup, retenção e archiving de tabelas de histórico e outbox planejadas?
- Há necessidade de réplicas de leitura para relatórios ou o tráfego atual é absorvido pela instância única?

## Related Potential ADRs
- [ORM Prisma para Acesso a Dados e Modelagem de Esquema](./orm-prisma.md)
- [Padrão Transacional Outbox no MySQL](../WEBHOOKS/padrao-transacional-outbox-no-mysql.md)

## Additional Notes
Classificado automaticamente como `must-document` pela Categoria 1 (Serviços de Infraestrutura - Step 0 da skill).
