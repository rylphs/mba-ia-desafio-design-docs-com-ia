# Potential ADR: Garantia de Entrega At-Least-Once com Desduplicação por Event ID

**Module**: WEBHOOKS
**Category**: Architecture
**Priority**: Must Document (Score: 140 / Override: docs/adrs/must-include.md)
**Date Identified**: 2026-09-13

---

## Existing ADR Context

ℹ️ **RELATED DECISIONS**

This decision relates to:
- **Padrão Transacional Outbox no MySQL** (`WEBHOOKS`): Onde o `event_id` único é gerado e persistido no momento da criação do evento.
- **Política de Retry com Backoff Exponencial e Tabela DLQ Dedicada** (`WEBHOOKS`): Cenário de retentativas onde duplicatas podem ser geradas se o cliente processar mas responder com timeout.

**When creating formal ADR**: Reference these in Related ADRs section.

---

## What Was Identified

Identificou-se a decisão de adotar a semântica de entrega *At-Least-Once* (pelo menos uma vez) para os webhooks de alteração de pedidos, acompanhada do envio de um identificador único universal de evento gerado na outbox no cabeçalho HTTP `X-Event-Id` (UUID v4).

Em sistemas distribuídos com comunicação HTTP sujeita a falhas transitórias de rede, timeouts e retentativas, é impossível garantir *Exactly-Once* sem um protocolo de consenso distribuído bilateral pesado e inviável entre sistemas heterogêneos. Como consequência de retentativas automáticas, um cliente B2B pode vir a receber o mesmo evento mais de uma vez.

Para mitigar os impactos de duplicidade, a plataforma atribui a responsabilidade de idempotência e desduplicação ao cliente consumidor, fornecendo os cabeçalhos contratuais padronizados:
- `X-Event-Id`: Identificador único do evento (UUID gerado no momento da gravação na outbox).
- `X-Webhook-Id`: Identificador da subscrição de webhook do cliente.
- `X-Timestamp`: Timestamp ISO 8601 do envio da notificação (útil também para prevenção contra ataques de repetição).

## Why This Might Deserve an ADR

- **Impact**: Estabelece o contrato de garantia de entrega de dados entre a plataforma e todos os clientes integradores B2B externos, prevenindo inconsistências causadas por processamento repetido de transições de status.
- **Trade-offs**: Exige que os clientes externos implementem controle de estado e desduplicação em suas pontas receptoras com base no `X-Event-Id`. Em contrapartida, simplifica enormemente a arquitetura interna do OMS e alinha o sistema aos padrões consolidados da indústria (ex: Stripe, GitHub, Twilio).
- **Complexity**: Baixa na plataforma (geração de UUID no momento do outbox e propagação nos headers HTTP). Média para a documentação e portal de desenvolvedores.
- **Team Knowledge**: Alto. O time de engenharia e produto deve estar totalmente alinhado com essa semântica para responder a dúvidas de clientes sobre reenvios.
- **Future Implications**: Torna a integração resiliente a partições de rede e garante que nenhum evento seja perdido (*no event loss*), mesmo que seja entregue mais de uma vez.

## Evidence Found in Codebase

### Key Files
- [`TRANSCRICAO.md`](file:///home/raphael/workspace/pos-ia/desafios/mba-ia-desafio-design-docs-com-ia/TRANSCRICAO.md#L146-L158) - Linhas 146-158 e 260-267
  - Diego, Sofia e Marcos acordando a garantia at-least-once com header `X-Event-Id` e delegação de idempotência ao cliente consumidor.
- [`docs/adrs/must-include.md`](file:///home/raphael/workspace/pos-ia/desafios/mba-ia-desafio-design-docs-com-ia/docs/adrs/must-include.md#L7) - Linha 7
  - Registro da decisão obrigatória na síntese técnica da reunião.
- [`package.json`](file:///home/raphael/workspace/pos-ia/desafios/mba-ia-desafio-design-docs-com-ia/package.json#L32) - Linha 32
  - Dependência oficial do pacote `uuid` (v11.0.3) já integrado à aplicação.

### Code Evidence
```typescript
// Contrato de cabeçalhos acordado para o envio HTTP pelo worker:
const headers = {
  'Content-Type': 'application/json',
  'X-Event-Id': event.id, // UUID único persistido no outbox
  'X-Webhook-Id': subscription.id,
  'X-Timestamp': new Date().toISOString(),
  'X-Signature': calculateHmac(event.payload, subscription.secret),
};
```

### Impact Analysis
- Introduced: Discutido e validado em 2026-09-13.
- Modified: Incorporado na especificação técnica de integração.
- Last change: 2026-09-13.
- Affects: Contratos públicos da API externa, worker de envio e documentação de integração do desenvolvedor B2B.
- Recent themes: "at-least-once", "X-Event-Id", "idempotência no cliente", "semântica de mensageria".

### Alternatives (if observable)
- **Exactly-Once Delivery**: Descartado por Diego e Sofia devido à inviabilidade técnica de coordenação bilateral em HTTP sem introduzir latência intolerável e complexidade excessiva de Two-Phase Commit distribuído.
- **At-Most-Once Delivery (Fire and Forget sem retries)**: Descartado categoricamente porque perder notificações de faturamento ou envio de pedidos quebra o requisito de negócio dos clientes B2B.

## Questions to Address in ADR (if created)

- Por quanto tempo é recomendado que os clientes retenham o cache/histórico de `X-Event-Id` para desduplicação (ex: 24h a 7 dias)?
- O payload deve conter o `event_id` também no corpo JSON além do cabeçalho `X-Event-Id`?
- Como orientar os clientes sobre o tratamento de falhas em sua camada de recepção?

## Related Potential ADRs
- [Padrão Transacional Outbox no MySQL](./padrao-transacional-outbox-no-mysql.md)
- [Política de Retry com Backoff Exponencial e Tabela DLQ Dedicada](./politica-retry-backoff-exponencial-tabela-dlq.md)
- [Autenticação e Integridade via HMAC-SHA256 com Secret por Endpoint](./autenticacao-integridade-hmac-sha256-secret-por-endpoint.md)

## Additional Notes
A decisão atende o item 3 do arquivo `docs/adrs/must-include.md` e está classificada obrigatoriamente como `must-document`.
