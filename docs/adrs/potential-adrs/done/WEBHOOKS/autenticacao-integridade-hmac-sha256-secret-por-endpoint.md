# Potential ADR: Autenticação e Integridade via HMAC-SHA256 com Secret por Endpoint

**Module**: WEBHOOKS
**Category**: Security
**Priority**: Must Document (Score: 140 / Override: docs/adrs/must-include.md)
**Date Identified**: 2026-09-13

---

## Existing ADR Context

ℹ️ **RELATED DECISIONS**

This decision relates to:
- **Garantia de Entrega At-Least-Once com Desduplicação por Event ID** (`WEBHOOKS`): O cabeçalho `X-Signature` trafega em conjunto com `X-Event-Id` e `X-Timestamp`.
- **Autenticação Stateless com JWT e RBAC** (`AUTH`): Modelo de segurança adotado para a API interna, que se complementa com a segurança de integração externa via HMAC.

**When creating formal ADR**: Reference these in Related ADRs section.

---

## What Was Identified

Identificou-se a decisão de adotar assinaturas criptográficas *HMAC-SHA256* para autenticação e verificação de integridade dos payloads de webhooks enviados aos clientes, utilizando um segredo criptográfico compartilhado (*shared secret*) exclusivo por endpoint cadastrado, com suporte a rotação de chaves com janela de carência (*grace period*) de 24 horas.

Como os webhooks trafegam para servidores fora da rede privada da empresa pela internet pública, é mandatório garantir que o receptor consiga atestar matematicamente que a requisição partiu legitimamente do OMS e que o conteúdo do payload não foi adulterado em trânsito (*tampering*).

As diretrizes de segurança acordadas incluem:
1. **Algoritmo Padrão de Mercado**: `HMAC-SHA256` calculado sobre o corpo bruto (`raw JSON body`) da requisição HTTP e transmitido no cabeçalho `X-Signature`.
2. **Segregação de Chaves**: Cada endpoint de webhook possui uma secret única gerada de forma criptograficamente segura (32 bytes em hexadecimal). Não há secret global da plataforma.
3. **Rotação com Grace Period**: Ao solicitar a rotação de secret pela API, a chave anterior permanece válida por até 24 horas simultaneamente à nova chave, concedendo tempo hábil para que os clientes atualizem suas configurações sem perda de eventos.
4. **Transporte Seguro Obrigatório**: TLS obrigatório (rejeição de URLs `http://` no schema de validação).

## Why This Might Deserve an ADR

- **Impact**: Protege clientes B2B contra ataques de personificação (*spoofing*), espionagem e adulteração maliciosa de dados de pedidos em trânsito, sem demandar infraestrutura pesada de certificados corporativos.
- **Trade-offs**: A secret por endpoint e rotação de 24 horas demandam suporte no modelo de dados para armazenar tanto a chave atual quanto a chave prévia (`previousSecret`) com timestamp de expiração (`previousSecretExpiresAt`). Em troca, o risco de vazamento de uma secret fica restrito a um único endpoint.
- **Complexity**: Média. Envolve uso do módulo nativo `crypto` do Node.js, geração segura de tokens e lógica de assinatura e rotação.
- **Team Knowledge**: Crítico. O time de segurança e engenharia precisa garantir que a assinatura seja gerada com precisão de bytes sobre o payload JSON.
- **Future Implications**: Estabelece o padrão de segurança para qualquer futura integração outbound ou webhook emitido pela plataforma.

## Evidence Found in Codebase

### Key Files
- [`TRANSCRICAO.md`](/TRANSCRICAO.md#L117-L145) - Linhas 117-145 e 273-276
  - Discussão detalhada liderada por Sofia (Engenheira de Segurança) definindo HMAC-SHA256, secret por endpoint, TLS obrigatório, grace period de 24h e agendamento de revisão de segurança.
- [`docs/adrs/must-include.md`](/docs/adrs/must-include.md#L9) - Linha 9
  - Registro da decisão obrigatória na síntese da reunião.
- [`src/config/env.ts`](/src/config/env.ts#L1-L20) - Linhas 1-20
  - Padrão de gestão de variáveis de ambiente e segurança.

### Code Evidence
```typescript
// Implementação padrão de assinatura HMAC-SHA256 com Node.js crypto:
import crypto from 'node:crypto';

export function signPayload(payload: string, secret: string): string {
  return crypto
    .createHmac('sha256', secret)
    .update(payload, 'utf8')
    .digest('hex');
}
```

### Impact Analysis
- Introduced: Definido em 2026-09-13 pela Engenharia de Segurança.
- Modified: Integrado aos contratos e modelos de persistência.
- Last change: 2026-09-13.
- Affects: Worker de envio HTTP, CRUD de webhooks, esquemas Zod (validação de URL HTTPS) e módulo de segurança.
- Recent themes: "HMAC-SHA256", "secret por endpoint", "rotação com grace period 24h", "TLS obrigatório".

### Alternatives (if observable)
- **Secret Global Compartilhada da Plataforma**: Rejeitada energicamente por Sofia e Diego devido ao risco de segurança sistêmico (vazamento por um cliente comprometeria todos os demais).
- **mTLS (Mutual TLS) ou Assinatura com Chaves Asimétricas (RSA/ECDSA)**: Descartados pela sobrecarga operacional de emissão e renovação de certificados para clientes de médio porte.
- **Rotação Imediata sem Carência**: Descartada por provocar interrupção no processamento do cliente durante a troca de chaves.

## Questions to Address in ADR (if created)

- O cabeçalho `X-Signature` deve ter prefixo de versão (ex: `sha256=...` ou apenas a hash hexadecimal pura)?
- Como garantir que a ordenação e espaços do JSON assinado correspondam exatamente ao que trafega na rede (snapshot serializado em string)?
- Qual deve ser a entropia para a geração do segredo (ex: `crypto.randomBytes(32).toString('hex')`)?

## Related Potential ADRs
- [Garantia de Entrega At-Least-Once com Desduplicação por Event ID](./garantia-entrega-at-least-once-com-desduplicacao-event-id.md)
- [Padrão Transacional Outbox no MySQL](./padrao-transacional-outbox-no-mysql.md)
- [Autenticação Stateless com JWT e RBAC](../../must-document/AUTH/autenticacao-jwt-e-rbac.md)

## Additional Notes
A decisão atende o item 5 do arquivo `docs/adrs/must-include.md` e está classificada obrigatoriamente como `must-document`.
