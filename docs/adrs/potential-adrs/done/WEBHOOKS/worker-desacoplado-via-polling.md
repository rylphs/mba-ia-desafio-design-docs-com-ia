# Potential ADR: Worker Desacoplado via Polling

**Module**: WEBHOOKS
**Category**: Architecture
**Priority**: Must Document (Score: 135 / Override: docs/adrs/must-include.md)
**Date Identified**: 2026-09-13

---

## Existing ADR Context

ℹ️ **RELATED DECISIONS**

This decision relates to:
- **Padrão Transacional Outbox no MySQL** (`WEBHOOKS`): Fonte de dados que o worker consultará via polling.
- **Política de Retry com Backoff Exponencial e Tabela DLQ Dedicada** (`WEBHOOKS`): Lógica de retentativas executada pelo worker.
- **Framework Web Express.js** (`INFRA`): O servidor web cujo ciclo de vida de processo permanece isolado deste worker.

**When creating formal ADR**: Reference these in Related ADRs section.

---

## What Was Identified

Identificou-se a decisão de implementar o consumo e processamento assíncrono dos eventos da outbox através de um worker em processo Node.js desacoplado (`src/worker.ts`), executando um loop de *polling* a cada 2 segundos no banco de dados MySQL.

O worker funcionará como um entrypoint independente da aplicação HTTP principal (`src/server.ts`), com seu próprio ciclo de vida, script de inicialização (`npm run worker`), gerenciamento de sinais de encerramento gracioso (`SIGINT`, `SIGTERM`) e pool de conexões com o banco gerenciado por uma instância dedicada do `PrismaClient`.

Para o cenário inicial, a arquitetura adotará um modelo *single-worker*, onde a ordenação dos eventos de um mesmo pedido é mantida pela leitura em lote (`batch`) ordenada por `createdAt ASC` da tabela `webhook_outbox`. O intervalo de polling de 2 segundos foi selecionado para assegurar que a latência de entrega permaneça confortavelmente abaixo do teto de 10 segundos exigido pelos clientes B2B.

## Why This Might Deserve an ADR

- **Impact**: Isola completamente o ciclo de vida do envio assíncrono de notificações do tráfego HTTP da API REST, prevenindo que picos de chamadas externas de webhooks consumam os recursos da API ou que reinicializações da API derrubem o processamento assíncrono.
- **Trade-offs**: A abordagem por polling em intervalos regulares (2s) gera consultas constantes ao banco mesmo em momentos de ociosidade (*polling overhead*). A garantia de ordenação por pedido é válida enquanto houver uma única instância ativa de worker.
- **Complexity**: Baixa a média. Requer um script de bootstrap em `src/worker.ts`, loop assíncrono seguro com tratamento de erros não capturados e queries otimizadas por índice no MySQL.
- **Team Knowledge**: Alto. Engenheiros de infraestrutura e backend precisam saber que a aplicação é composta por dois processos distintos em tempo de execução: o servidor HTTP e o worker de webhooks.
- **Future Implications**: Caso o volume aumente e demande múltiplos workers, a arquitetura precisará evoluir para particionamento de pedidos, locks pessimistas (`SELECT ... FOR UPDATE SKIP LOCKED`) ou migração para fila distribuída.

## Evidence Found in Codebase

### Key Files
- [`src/server.ts`](/src/server.ts#L1-L28) - Linhas 1-28
  - Estrutura de bootstrap do processo HTTP atual, que servirá de referência de padrão para `src/worker.ts`.
- [`package.json`](/package.json#L10-L21) - Linhas 10-21
  - Scripts NPM onde o comando `worker` será configurado (`"worker": "tsx watch --env-file=.env src/worker.ts"`).
- [`src/config/database.ts`](/src/config/database.ts#L1-L10) - Linhas 1-10
  - Configuração do `PrismaClient` a ser instanciado de forma isolada por processo.
- [`TRANSCRICAO.md`](/TRANSCRICAO.md#L59-L88) - Linhas 59-88
  - Diego e Larissa decidindo pelo worker separado em polling de 2 segundos, descartando triggers de banco e aceitando a restrição de single-worker inicial.
- [`docs/adrs/must-include.md`](/docs/adrs/must-include.md#L6) - Linha 6
  - Requisito mandatório de documentação técnica.

### Code Evidence
```typescript
// Padrão de bootstrap identificado em src/server.ts:6-22 a ser replicado em src/worker.ts
async function bootstrap(): Promise<void> {
  // Inicialização independente de processo
  const shutdown = async (signal: string): Promise<void> => {
    logger.info({ signal }, 'shutdown_initiated');
    // Encerramento limpo do loop e desconexão do Prisma
    await prisma.$disconnect();
    process.exit(0);
  };

  process.on('SIGINT', () => void shutdown('SIGINT'));
  process.on('SIGTERM', () => void shutdown('SIGTERM'));
}
```

### Impact Analysis
- Introduced: Definido na reunião técnica em 2026-09-13.
- Modified: Documentado a partir dos padrões da codebase (`7ef4317`).
- Last change: 2026-09-13.
- Affects: Entrypoints do sistema (`package.json`, scripts de container), módulo `WEBHOOKS` e infraestrutura de deploy.
- Recent themes: "worker desacoplado", "processo separado", "polling 2s", "single-worker ordering".

### Alternatives (if observable)
- **Worker Embutido no Processo da API Express**: Descartado por Diego e Larissa para evitar acoplamento de recursos, concorrência no event loop da API e indisponibilidade de envios durante restarts de deploy.
- **Triggers de Banco com Notificação Externa**: Descartado por Diego pela ausência de listener nativo reativo no MySQL (como NOTIFY/LISTEN do PostgreSQL) e complexidade de integração segura.

## Questions to Address in ADR (if created)

- Qual deve ser o tamanho máximo do lote (`batch size`) lido em cada ciclo de polling (ex: 20 a 50 eventos)?
- Como evitar acúmulo de requisições concorrentes se um ciclo de envio demorar mais que o intervalo de 2s?
- Como orquestrar o grace period no encerramento (`SIGTERM`) para não abortar envios HTTP em andamento?

## Related Potential ADRs
- [Padrão Transacional Outbox no MySQL](./padrao-transacional-outbox-no-mysql.md)
- [Política de Retry com Backoff Exponencial e Tabela DLQ Dedicada](./politica-retry-backoff-exponencial-tabela-dlq.md)
- [Framework Web Express.js](../../must-document/INFRA/framework-web-express.md)

## Additional Notes
A decisão atende o item 2 do arquivo `docs/adrs/must-include.md` e está classificada obrigatoriamente como `must-document`.
