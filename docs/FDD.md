### FDD: Sistema de Webhooks de Notificacao de Pedidos

Versao: 1.0.0  
Data: 2026-09-15  
Responsavel: Bruno (Engenheiro Pleno - Time de Pedidos) e Diego (Engenheiro Senior - Time de Plataforma)  

---

### 1. Contexto e motivacao tecnica

A plataforma de gerenciamento de pedidos (Order Management System - OMS) opera atualmente sem nenhum mecanismo nativo de notificacao assincrona de eventos para sistemas externos. Clientes corporativos B2B prioritarios (Atlas Comercial, MaxDistribuidora e Nova Cargo) necessitam acompanhar em tempo real o ciclo de vida de seus pedidos e realizam consultas constantes via varredura (polling ativo) no endpoint `GET /orders`. Esse padrao gera sobrecarga severa de CPU na API Express, consumo excessivo de conexoes no pool do banco relacional MySQL e latencia elevada para integradores externos. Alem disso, ha risco iminente de perda de contrato (churn) com a Atlas Comercial caso uma esteira de notificacoes com latencia inferior a 10 segundos nao seja entregue ate o final do trimestre vigente (novembro de 2026).

No High-Level Design (HLD), a mudanca de estado do pedido e orquestrada pelo metodo `changeStatus` em `OrderService`. Esse metodo executa uma transacao relacional no MySQL que atualiza a tabela `orders`, registra a auditoria em `order_status_history` e ajusta o estoque em `stock_quantity`. Nao e viavel realizar disparos HTTP sincronos dentro dessa transacao devido ao risco de lentidao externa reter locks de banco, travar threads do event loop do Node.js e induzir cenarios de falha em cascata. Tambem nao e viavel publicar diretamente em brokers externos sem coordenacao transacional devido ao problema da escrita dupla (dual-write problem).

A solucao tecnica consiste na implementacao do Padrao Transacional Outbox integrado atomicamente a transacao SQL de `changeStatus`. Um snapshot congelado dos dados do evento e persistido na tabela `webhook_outbox`. Um worker assincrono desacoplado, executando como processo dedicado (`src/worker.ts`), realiza consultas periodicas (polling) a cada 2 segundos, despacha as chamadas HTTP assinadas com HMAC-SHA256, gerencia retentativas com backoff exponencial e segrega falhas exauridas em uma tabela de Dead Letter Queue (`webhook_dead_letter`).

**Atores do sistema:**
- Operador / API Client: Usuario autenticado que solicita a transicao de estado do pedido via API.
- Cliente B2B (Receptor): Servidor externo HTTPS configurado pelo parceiro comercial para receber os eventos.
- Worker de Webhooks: Processo em background responsavel pela leitura da outbox, calculo criptografico e despacho HTTP.
- Administrador do Sistema: Usuario autenticado com papel ADMIN autorizado a reprocessar eventos retidos na DLQ.

**Limites do escopo:**
- A feature cobre exclusivamente o fluxo de saida de notificacoes (outbound webhooks).
- Nao contempla interfaces visuais de usuario (painel/dashboard).
- Nao realiza envio de alertas por canais alternativos (e-mail ou SMS).
- Nao adota plataformas externas de mensageria (Kafka, RabbitMQ ou Redis Cluster), preservando a infraestrutura MySQL existente.

---

### 2. Objetivos tecnicos

- **Latencia de Entrega P99 < 10s:** Assegurar que 99% das notificacoes de alteracao de status sejam entregues no endpoint de destino em menos de 10 segundos a partir do commit da transacao do pedido.
- **Consistencia Transacional Atomica (Zero Dual-Write):** Garantir invariante de que nenhum evento pendente seja criado sem o commit da transacao do pedido, e que nenhuma alteracao confirmada deixe de gerar seu respectivo registro na outbox.
- **Garantia de Entrega At-Least-Once com Desduplicacao:** Garantir que todo evento seja despachado pelo menos uma vez, disponibilizando o cabecalho imutavel `X-Event-Id` (UUID v4) para desduplicacao idempotente na ponta receptora.
- **Resiliencia com Janela de 15 Horas:** Executar politica de 5 retentativas progressivas com backoff exponencial (1m, 5m, 30m, 2h, 12h) acrescido de variacao aleatoria (jitter de 20%), tolerando janelas estendidas de indisponibilidade de parceiros antes do descarte para DLQ.
- **Autenticidade e Integridade Criptografica:** Assinar 100% dos payloads enviados utilizando HMAC-SHA256 com chave secreta unica por endpoint, com cabecalho `X-Timestamp` para prevencao de ataques de repeticao e suporte a rotacao de chaves com carencia de 24 horas.
- **Isolamento Total de Falhas Externas:** Invariante de que indisponibilidade, lentidao extrema ou falhas remotas de clientes B2B nao causem lentidao, rollback ou degradacao no tempo de resposta da API principal de pedidos.

---

### 3. Escopo e exclusoes

**Incluido**
- Modelagem e migracao Prisma das tabelas `webhook_outbox`, `webhook_dead_letter` e configuracao de endpoints de clientes.
- Integracao atomica da funcao `publishWebhookEvent` na transacao existente em `OrderService.changeStatus`.
- Implementacao do worker assincrono desacoplado em processo separado (`src/worker.ts`) com ciclo de polling de 2 segundos.
- Politica automatica de retry com backoff exponencial (1m, 5m, 30m, 2h, 12h) e full jitter aleatorio.
- Segregacao de eventos falhos em tabela dedicada de DLQ (`webhook_dead_letter`).
- Modulo REST completo em `src/modules/webhooks` com endpoints de CRUD de configuracao de endpoints e historico de entregas (`/webhooks/:id/deliveries`).
- Endpoint administrativo protegido por perfil ADMIN para reprocessamento manual de eventos da DLQ (`/admin/webhooks/dead-letter/:id/replay`).
- Assinatura digital HMAC-SHA256 no cabecalho `X-Signature` e metadados de seguranca (`X-Timestamp`, `X-Event-Id`, `X-Webhook-Id`).
- Endpoint de rotacao de secret com periodo de carencia de 24 horas para convivencia de chaves.
- Validacao estrita de esquemas de entrada com Zod, exigindo transporte seguro HTTPS e rejeitando payloads superiores a 64KB.

**Excluido**
- Recepcao de webhooks de terceiros (inbound webhooks).
- Painel ou interface grafica para visualizacao de webhooks (escopo exclusivo de frontend).
- Notificacao proativa de suporte ou alertas por e-mail em caso de falhas consecutivas de clientes.
- Mecanismos de controle de vazao de saida (outbound rate limiting) na fase inicial de lancamento.
- Garantia de ordenacao global irrestrita entre pedidos distintos (ordenacao mantida estritamente por pedido via single-worker).
- Provedores externos de mensageria (RabbitMQ, Apache Kafka, Redis Streams ou servicos gerenciados de cloud).

---

### 4. Fluxos detalhados e diagramas

#### Fluxo principal (Criacao do Pedido, Outbox e Despacho com Sucesso)
1. O operador ou servico externo envia requisicao `PATCH /orders/:id/status` para a API Express.
2. A requisicao passa pelo middleware de autenticacao JWT e validacao de esquema Zod.
3. `OrderService.changeStatus` abre transacao atomica no MySQL via `prisma.$transaction`.
4. A transacao atualiza o registro do pedido em `orders`, insere o historico em `order_status_history` e atualiza o estoque.
5. Dentro do mesmo escopo transacional `tx`, a funcao `publishWebhookEvent` busca os webhooks ativos do cliente configurados para aquele status de destino.
6. Se houver correspondencia, serializa o snapshot imutavel do pedido em JSON e insere o registro na tabela `webhook_outbox` com `status = 'PENDING'`, `event_id = UUID()`, `retry_count = 0` e `next_retry_at = NOW()`.
7. A transacao SQL e confirmada (COMMIT). A resposta HTTP 200 e devolvida imediatamente ao cliente da API de pedidos.
8. O processo `src/worker.ts`, operando em loop continuo com tick a cada 2 segundos, executa consulta indexada na tabela `webhook_outbox` buscando lotes pendentes (`status = 'PENDING' AND next_retry_at <= NOW()`).
9. O worker recupera a secret ativa do endpoint correspondente e calcula a assinatura digital HMAC-SHA256 sobre o corpo da requisicao concatenado com o timestamp.
10. O worker dispara requisicao HTTP POST para a URL HTTPS do cliente com timeout estrito de 10 segundos, injetando os cabecalhos `X-Event-Id`, `X-Signature`, `X-Timestamp` e `X-Webhook-Id`.
11. O endpoint do cliente responde com status HTTP 2xx dentro de 10 segundos.
12. O worker atualiza o registro na `webhook_outbox` para `status = 'DELIVERED'` e registra a evidencia no historico de entregas.

#### Fluxos alternativos e excecoes

**Variacao 1: Falha Transitoria do Cliente e Politica de Retentativa com Jitter**
1. O worker executa o disparo HTTP, mas o servidor do cliente retorna erro 5xx, 429 ou ocorre timeout de conexao (10 segundos).
2. O worker intercepta o erro e verifica o contador de tentativas (`retry_count`).
3. Sendo `retry_count < 5`, o worker incrementa `retry_count` em 1 unidade.
4. O worker calcula o tempo de espera base conforme a tabela de progressao (1m, 5m, 30m, 2h, 12h) e aplica variacao aleatoria (jitter) de mais ou menos 20%.
5. O worker atualiza a `webhook_outbox` definindo `status = 'PENDING'`, gravando o ultimo erro e agendando `next_retry_at` para o novo horario calculado.
6. No ciclo correspondente de polling, o worker retoma a mensagem e efetua nova tentativa.

**Variacao 2: Esgotamento de Tentativas e Movimentacao para DLQ**
1. A quinta retentativa falha (apos aproximadamente 15 horas de tentativas cumulativas).
2. O worker constata que `retry_count >= 5`.
3. O worker abre transacao SQL para mover o evento: insere o registro detalhado na tabela `webhook_dead_letter` (incluindo payload original, historico de status de resposta HTTP e mensagem de erro) e atualiza ou remove o registro da `webhook_outbox`.
4. O evento deixa de ser consultado pelo loop de polling regular, liberando a tabela operacional.

**Variacao 3: Replay Manual de Mensagem Morta via Administracao**
1. O suporte tecnico do cliente B2B normaliza seu endpoint e solicita o reprocessamento das notificacoes perdidas.
2. Um usuario administrador envia `POST /admin/webhooks/dead-letter/:id/replay`.
3. O middleware valida o token JWT e confirma o papel `ADMIN` via `requireRole('ADMIN')`.
4. O servico de administracao localiza o registro na tabela `webhook_dead_letter`.
5. Em transacao atomica, reinsere o evento na tabela `webhook_outbox` com `status = 'PENDING'`, `retry_count = 0`, `next_retry_at = NOW()` e carimba `replayed_at = NOW()` e `replayed_by = userId` na tabela de DLQ para fins de auditoria.
6. O worker de segundo plano captura o evento no tick seguinte de polling e executa o disparo.

**Variacao 4: Rotacao de Secret com Periodo de Carencia (Grace Period)**
1. O cliente solicita a geracao de nova secret via `POST /webhooks/:id/rotate-secret`.
2. O sistema gera nova credencial criptografica de 32 bytes (hexadecimal de 64 caracteres).
3. A secret antiga e movida para o campo `previous_secret` com validade definida em `previous_secret_expires_at = NOW() + 24 horas`.
4. Durante as proximas 24 horas, requisicoes podem ser assinadas com a nova secret, enquanto a anterior permanece disponivel para validacao de transicao pelo cliente. Decorrido o prazo, a secret legada e purgada.

#### Diagramas

```mermaid
sequenceDiagram
    autonumber
    actor Operador as Operador / API
    participant OrderSvc as OrderService.changeStatus
    participant DB as MySQL (ACID)
    participant Outbox as webhook_outbox
    participant Worker as Worker (src/worker.ts)
    participant Destino as Endpoint Cliente B2B
    participant DLQ as webhook_dead_letter

    Operador->>OrderSvc: PATCH /orders/:id/status
    activate OrderSvc
    OrderSvc->>DB: Inicia Transacao SQL
    OrderSvc->>DB: UPDATE orders, INSERT history, UPDATE stock
    OrderSvc->>Outbox: INSERT evento PENDING (snapshot JSON)
    OrderSvc->>DB: COMMIT da Transacao
    OrderSvc-->>Operador: 200 OK (Status Atualizado)
    deactivate OrderSvc

    loop Loop a cada 2 segundos
        Worker->>Outbox: SELECT WHERE status='PENDING' AND next_retry_at <= NOW() LIMIT 50
        Outbox-->>Worker: Retorna lote de eventos
        Worker->>Destino: POST /webhook (Headers: HMAC, Event-Id, Timestamp)
        alt Resposta 200 OK dentro de 10s
            Destino-->>Worker: 200 OK
            Worker->>Outbox: UPDATE status = 'DELIVERED'
        else Falha ou Timeout (Tentativas < 5)
            Destino--xWorker: 5xx, Timeout ou Erro de Rede
            Worker->>Outbox: UPDATE retry_count = retry_count + 1, next_retry_at = NOW() + Backoff com Jitter
        else Falha na 5a tentativa (Esgotado)
            Destino--xWorker: 5xx ou Timeout persistente
            Worker->>DLQ: INSERT INTO webhook_dead_letter (...)
            Worker->>Outbox: UPDATE status = 'FAILED'
        end
    end
```

---

### 5. Contratos publicos (assinaturas, endpoints, headers, exemplos)

#### [Contrato 1] Cadastro de Endpoint de Webhook
- Tipo: endpoint
- Assinatura/Rota: POST /webhooks
- Metodo: POST
- Semantica de status/headers:
  - 201 Created: Webhook cadastrado com sucesso. Retorna credencial secreta gerada.
  - 400 Bad Request: Dados invalidos (ex: URL nao e HTTPS, formato invalido de URL ou eventos nao suportados).
  - 401 Unauthorized: Ausencia de token JWT ou token invalido.
  - Header Authorization: Formato Bearer [token_jwt].

**Exemplo de requisicao**
```json
{
  "customerId": "9b1deb4d-3b7d-4bad-9bdd-2b0d7b3dcb6d",
  "url": "https://api.atlascomercial.com.br/v1/integracao/pedidos",
  "events": [
    "PAID",
    "PROCESSING",
    "SHIPPED",
    "DELIVERED",
    "CANCELLED"
  ],
  "description": "Notificacoes de status de pedidos para ERP Atlas"
}
```

**Exemplo de resposta**
```json
{
  "id": "e4b3c2a1-5d6e-7f8a-9b0c-1d2e3f4a5b6c",
  "customerId": "9b1deb4d-3b7d-4bad-9bdd-2b0d7b3dcb6d",
  "url": "https://api.atlascomercial.com.br/v1/integracao/pedidos",
  "secret": "whsec_8f4b1d7a3e9c205f61728394a5b6c7d8e9f01a2b3c4d5e6f7a8b9c0d1e2f3a4b",
  "events": [
    "PAID",
    "PROCESSING",
    "SHIPPED",
    "DELIVERED",
    "CANCELLED"
  ],
  "active": true,
  "createdAt": "2026-09-15T14:30:00.000Z",
  "updatedAt": "2026-09-15T14:30:00.000Z"
}
```

---

#### [Contrato 2] Listagem de Webhooks Cadastrados
- Tipo: endpoint
- Assinatura/Rota: GET /webhooks
- Metodo: GET
- Semantica de status/headers:
  - 200 OK: Lista de webhooks configurados. O campo secret e mascarado por seguranca.
  - 401 Unauthorized: Ausencia ou invalidade de credencial JWT.
  - Query Param customerId (obrigatorio): Filtra os registros pelo identificador do cliente.

**Exemplo de requisicao**
```text
GET /webhooks?customerId=9b1deb4d-3b7d-4bad-9bdd-2b0d7b3dcb6d HTTP/1.1
Host: api.oms.empresa.com.br
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

**Exemplo de resposta**
```json
{
  "data": [
    {
      "id": "e4b3c2a1-5d6e-7f8a-9b0c-1d2e3f4a5b6c",
      "customerId": "9b1deb4d-3b7d-4bad-9bdd-2b0d7b3dcb6d",
      "url": "https://api.atlascomercial.com.br/v1/integracao/pedidos",
      "secretMasked": "whsec_8f4b...3a4b",
      "events": [
        "SHIPPED",
        "DELIVERED"
      ],
      "active": true,
      "createdAt": "2026-09-15T14:30:00.000Z",
      "updatedAt": "2026-09-15T14:30:00.000Z"
    }
  ]
}
```

---

#### [Contrato 3] Atualizacao Parcial de Webhook
- Tipo: endpoint
- Assinatura/Rota: PATCH /webhooks/:id
- Metodo: PATCH
- Semantica de status/headers:
  - 200 OK: Configuracao atualizada com sucesso.
  - 400 Bad Request: Dados invalidos no corpo da requisicao.
  - 404 Not Found: Identificador de webhook inexistente.

**Exemplo de requisicao**
```json
{
  "url": "https://novo-endpoint.atlascomercial.com.br/webhooks",
  "events": [
    "SHIPPED",
    "DELIVERED",
    "CANCELLED"
  ],
  "active": true
}
```

**Exemplo de resposta**
```json
{
  "id": "e4b3c2a1-5d6e-7f8a-9b0c-1d2e3f4a5b6c",
  "customerId": "9b1deb4d-3b7d-4bad-9bdd-2b0d7b3dcb6d",
  "url": "https://novo-endpoint.atlascomercial.com.br/webhooks",
  "secretMasked": "whsec_8f4b...3a4b",
  "events": [
    "SHIPPED",
    "DELIVERED",
    "CANCELLED"
  ],
  "active": true,
  "updatedAt": "2026-09-15T15:45:10.000Z"
}
```

---

#### [Contrato 4] Consulta de Historico de Entregas
- Tipo: endpoint
- Assinatura/Rota: GET /webhooks/:id/deliveries
- Metodo: GET
- Semantica de status/headers:
  - 200 OK: Retorna as ultimas 100 tentativas de entrega para o webhook solicitado.
  - 404 Not Found: Webhook nao encontrado.
  - Query Param limit: Opcional, default 50, maximo 100.

**Exemplo de requisicao**
```text
GET /webhooks/e4b3c2a1-5d6e-7f8a-9b0c-1d2e3f4a5b6c/deliveries?limit=2 HTTP/1.1
Host: api.oms.empresa.com.br
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

**Exemplo de resposta**
```json
{
  "data": [
    {
      "id": "7a8b9c0d-1e2f-3a4b-5c6d-7e8f9a0b1c2d",
      "webhookId": "e4b3c2a1-5d6e-7f8a-9b0c-1d2e3f4a5b6c",
      "eventId": "f1e2d3c4-b5a6-7890-1234-56789abcdef0",
      "eventType": "order.status_changed",
      "statusCode": 200,
      "status": "DELIVERED",
      "attempt": 1,
      "durationMs": 185,
      "requestHeaders": {
        "X-Event-Id": "f1e2d3c4-b5a6-7890-1234-56789abcdef0",
        "X-Timestamp": "1789482600"
      },
      "createdAt": "2026-09-15T15:50:00.185Z"
    }
  ]
}
```

---

#### [Contrato 5] Replay Administrativo de Mensagem Morta (DLQ)
- Tipo: endpoint
- Assinatura/Rota: POST /admin/webhooks/dead-letter/:id/replay
- Metodo: POST
- Semantica de status/headers:
  - 200 OK: Evento reenfileirado na outbox com status PENDING para reprocessamento imediato.
  - 401 Unauthorized: Nao autenticado.
  - 403 Forbidden: Usuario nao possui role ADMIN (validado por requireRole).
  - 404 Not Found: Registro de DLQ inexistente.

**Exemplo de requisicao**
```text
POST /admin/webhooks/dead-letter/c9b8a7f6-e5d4-3c2b-1a0f-9e8d7c6b5a4f/replay HTTP/1.1
Host: api.oms.empresa.com.br
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9... (Token de Usuario ADMIN)
```

**Exemplo de resposta**
```json
{
  "message": "Dead letter event re-enqueued successfully",
  "deadLetterId": "c9b8a7f6-e5d4-3c2b-1a0f-9e8d7c6b5a4f",
  "replayedOutboxId": "a1b2c3d4-e5f6-7a8b-9c0d-1e2f3a4b5c6d",
  "eventId": "f1e2d3c4-b5a6-7890-1234-56789abcdef0",
  "replayedBy": "usr-admin-001",
  "replayedAt": "2026-09-15T16:00:00.000Z"
}
```

---

#### [Contrato 6] Despacho de Notificacao de Webhook (Chamada Outbound do Worker)
- Tipo: endpoint
- Assinatura/Rota: POST [URL_DO_CLIENTE_HTTPS]
- Metodo: POST
- Semantica de status/headers:
  - Header X-Event-Id: UUID v4 unico por evento para desduplicacao idempotente.
  - Header X-Signature: Assinatura hexadecimal HMAC-SHA256 sobre o corpo da requisicao.
  - Header X-Timestamp: Unix Epoch em segundos no momento do despacho para prevencao de replay attack.
  - Header X-Webhook-Id: UUID da configuracao do webhook.
  - Header Content-Type: application/json.
  - Resposta esperada: HTTP 200 OK em ate 10 segundos.

**Exemplo de requisicao**
```json
{
  "eventId": "f1e2d3c4-b5a6-7890-1234-56789abcdef0",
  "eventType": "order.status_changed",
  "timestamp": "2026-09-15T14:35:00.000Z",
  "data": {
    "orderId": "d3b07384-d113-4a8b-967a-12e022f6d289",
    "orderNumber": "ORD-000128",
    "customerId": "9b1deb4d-3b7d-4bad-9bdd-2b0d7b3dcb6d",
    "fromStatus": "PROCESSING",
    "toStatus": "SHIPPED",
    "totalCents": 154000
  }
}
```

**Exemplo de resposta esperada**
```json
{
  "received": true
}
```

---

### 6. Erros, excecoes e fallback

#### Matriz de erros operacionais previstos (Prefixo WEBHOOK_)

| Codigo de Erro | HTTP Status | Condicao Disparadora | Tratamento Aplicado |
| :--- | :---: | :--- | :--- |
| `WEBHOOK_NOT_FOUND` | 404 | Identificador de webhook nao encontrado no banco | Interrompe fluxo e retorna 404 estruturado |
| `WEBHOOK_INVALID_URL` | 400 | URL com formato sintatico invalido | Rejeita requisicao via schema Zod com detalhamento |
| `WEBHOOK_INSECURE_URL` | 400 | URL utiliza protocolo HTTP inseguro | Exige schema HTTPS estrito com rejeicao imediata |
| `WEBHOOK_INVALID_EVENT_FILTER` | 400 | Evento solicitado nao pertence ao enum OrderStatus | Retorna lista de status validos aceitos |
| `WEBHOOK_SECRET_REQUIRED` | 400 | Operacao exige presenca de secret valida | Solicita passagem ou regeneracao da chave |
| `WEBHOOK_PAYLOAD_TOO_LARGE` | 413 | Snapshot do evento excede o limite estrito de 64KB | Registra alerta no Pino, cancela envio e notifica operacao |
| `WEBHOOK_DEAD_LETTER_NOT_FOUND` | 404 | ID informado para replay nao existe na tabela de DLQ | Retorna 404 indicando inexistencia do registro |
| `WEBHOOK_DELIVERY_TIMEOUT` | 504 | Endpoint remoto nao responde dentro da janela de 10s | Registra falha de timeout e programa retry com jitter |
| `WEBHOOK_SIGNATURE_FAILED` | 500 | Falha no calculo criptografico da assinatura HMAC | Bloqueia envio, loga erro critico e retem na outbox |

#### Estrategias de resiliencia
- **Timeouts:** Timeout estrito de conexao e leitura de 10 segundos para toda chamada HTTP externa realizada pelo worker.
- **Retries e Backoff:** Progressao em 5 passos (1m, 5m, 30m, 2h, 12h) totalizando janela de ~15 horas.
- **Full Jitter:** Aplicacao de variacao aleatoria de 20% no calculo do proximo retry ($t_{\text{retry}} = \text{base} \pm \text{random}(0, 0.2 \times \text{base})$) para desincronizar ondas de requisicoes simultaneas.
- **Circuit Breaker Implicito via DLQ:** Suspensao automatica de envios para eventos exauridos, impedindo que destinos mortos degradem o rendimento do worker.

#### Politica de fallback
- Caso a insercao na `webhook_outbox` falhe durante a atualizacao do pedido, a transacao relacional inteira sofre rollback, garantindo que o status do pedido nao mude sem garantia de notificacao.
- Caso o endpoint do cliente esteja permanentemente indisponivel, o cliente tem como fallback de integracao a consulta manual ao endpoint existente `GET /orders/:id`.

#### Invariantes criticos
- Nenhum registro pode existir na tabela `webhook_outbox` sem que a correspondente alteracao em `orders` e `order_status_history` tenha sido commitada.
- O snapshot JSON serializado na insercao do evento e imutavel e nao deve ser recalculado durante as retentativas do worker.
- O cabecalho `X-Event-Id` e gerado uma unica vez por evento e permanece identico em todas as 5 tentativas de envio.

---

### 7. Observabilidade

#### Metricas essenciais
- `webhook_outbox_pending_count` (Gauge): Quantidade de eventos aguardando processamento na outbox ativa.
- `webhook_dispatch_duration_ms` (Histogram): Tempo de resposta das chamadas HTTP externas aos parceiros (buckets: 100ms, 500ms, 1s, 2s, 5s, 10s).
- `webhook_delivery_attempts_total` (Counter, labels: `status_code`, `customer_id`, `success`): Total de tentativas de despacho realizadas.
- `webhook_retries_scheduled_total` (Counter, labels: `attempt_number`): Total de retentativas agendadas.
- `webhook_dead_letter_total` (Counter, labels: `customer_id`): Quantidade de eventos movidos para a DLQ apos esgotamento.

#### Logs estruturados (Biblioteca Pino)
O modulo utiliza a instancia central de logger Pino em `src/shared/logger/index.ts`. Todos os eventos de log do worker contem obrigatoriamente os seguintes atributos estruturados:
- Formato: JSON com carimbo ISO 8601, level numérico/texto e mensagem.
- Atributos padronizados: `event_id`, `webhook_id`, `customer_id`, `attempt`, `duration_ms`, `http_status`, `target_url` e `worker_pid`.
- Mascaramento estrito: Campos de segredo (`secret`, `passwordHash`) sao estritamente suprimidos das saidas de log.

#### Tracing distribuido
- Spans principais:
  - `webhook.publish`: Criacao do registro outbox na transacao Prisma de pedidos.
  - `webhook.worker.poll`: Ciclo de varredura e selecao de lote no banco.
  - `webhook.dispatch`: Montagem de payload, calculo de HMAC e chamada HTTP via Axios/Fetch.
  - `webhook.retry.schedule`: Calculo de backoff com jitter e atualizacao de `next_retry_at`.
- Amostragem: 100% para eventos de falha e erro (status >= 400); 10% para despachos com sucesso 2xx.

#### Dashboards e alertas minimos
- **Alerta de Latencia P99:** Disparado se `webhook_dispatch_duration_ms{quantile="0.99"} > 8s` por mais de 5 minutos consecutivos.
- **Alerta de Fila Represada (Lag da Outbox):** Disparado se `webhook_outbox_pending_count > 500` ou se o evento mais antigo pendente possuir `NOW() - next_retry_at > 30s`.
- **Alerta de Entrada em DLQ:** Disparado imediatamente a cada incremento no contador `webhook_dead_letter_total`.

---

### 8. Dependencias e compatibilidade

| Componente | Versao minima | Observacoes |
| :--- | :--- | :--- |
| Node.js | >= 20.x LTS | Runtime de execucao da API e do Worker dedicado |
| TypeScript | >= 5.x | Compilacao tipada e checagem estatica |
| MySQL | >= 8.0 | Suporte a tipos JSON nativos e funcoes transacionais ACID |
| Prisma ORM | >= 5.x | Gerenciamento de conexoes, migrations e cliente tipado |
| Express | >= 4.19 | Framework HTTP da API REST |
| Pino | >= 8.x | Logger estruturado de alta performance |
| Zod | >= 3.x | Validacao de schemas declarativos de entrada e HTTPS |

**Garantias de compatibilidade**
- **Paridade de Banco de Dados:** Todas as consultas do worker utilizam recursos compativeis com o MySQL 8.x ja provisionado, sem extensões proprietarias.
- **Versionamento Semantico de Payloads:** Eventos utilizam versao explicita de esquema. O payload nao inclui arrays de itens (`items`) para evitar quebras por crescimento de schema. Se o cliente necessitar de detalhes, devera consumir `GET /orders/:id`.
- **Compatibilidade Retroativa:** A adicao da tabela outbox nao altera a estrutura de colunas das tabelas existentes (`orders`, `order_status_history`, `products`).

---

### 9. Criterios de aceite tecnicos

- [CA-01] O schema Prisma contem as tabelas `webhook_endpoints`, `webhook_outbox` e `webhook_dead_letter` com indices validados em `status`, `next_retry_at` e `created_at`.
- [CA-02] A alteracao de status do pedido em `OrderService.changeStatus` grava o evento na tabela `webhook_outbox` na mesma transacao SQL; havendo rollback do pedido, o evento nao e persistido.
- [CA-03] O worker assincrono roda como processo desacoplado via script `npm run worker` instanciando cliente Prisma proprio.
- [CA-04] O intervalo de polling do worker e de exatamente 2 segundos, processando no maximo 50 registros por lote.
- [CA-05] 99% dos eventos gerados em ambiente de teste sao entregues com latencia ponta a ponta estritamente inferior a 10 segundos.
- [CA-06] O cabecalho `X-Event-Id` e emitido com UUID v4 valido e permanece inalterado em retentativas consecutivas.
- [CA-07] A assinatura `X-Signature` e calculada via HMAC-SHA256 utilizando a secret do endpoint sobre o corpo da requisicao e o timestamp.
- [CA-08] Requisicoes com URL com protocolo HTTP sao rejeitadas no cadastro com codigo `WEBHOOK_INSECURE_URL` e status 400.
- [CA-09] Falhas de envio disparam politica de 5 retries com progressao de backoff exponencial acrescida de jitter de 20%.
- [CA-10] Apos 5 falhas consecutivas, o evento e transferido para a tabela `webhook_dead_letter` com motivo do erro documentado.
- [CA-11] O endpoint `POST /admin/webhooks/dead-letter/:id/replay` re-enfileira o evento na outbox e valida permissao restrita a role `ADMIN`.
- [CA-12] O endpoint `GET /webhooks/:id/deliveries` retorna o historico paginado com codigo de status HTTP, tempo de resposta e headers.

---

### 10. Riscos e mitigacao

### [Risco 1] Saturacao de Leituras no Banco de Dados por Polling Constante do Worker

- **Probabilidade:** Media
- **Impacto:** Alto
- **Mitigacao:**
  - Criacao de indice composto otimizado cobrindo exatamente os campos da consulta: `INDEX idx_outbox_polling (status, next_retry_at, created_at)`.
  - Limitacao estrita do tamanho do lote de leitura (`LIMIT 50`) para evitar scans de tabela.
  - Implementacao de rotina agendada de expurgo de registros com status `DELIVERED` apos 30 dias para evitar inchaco da arvore B-Tree.
- **Plano de contingencia:** Aumentar o intervalo de polling de 2s para 5s via variavel de ambiente configuravel sem necessidade de novo deploy.

### [Risco 2] Vazamento de Secret de Webhook por Falha Operacional do Cliente

- **Probabilidade:** Media
- **Impacto:** Alto
- **Mitigacao:**
  - Geracao de credenciais com alta entropia criptografica (32 bytes aleatorios em hexadecimal).
  - Isolamento rigoroso: cada endpoint de cada cliente possui uma chave secreta exclusiva.
  - Disponibilizacao de endpoint de rotacao de segredo com periodo de carencia de 24 horas (`POST /webhooks/:id/rotate-secret`).
  - Mascaramento obrigatorio das chaves nos endpoints de consulta (`secretMasked`).
- **Plano de contingencia:** Exclusao ou desativacao imediata do endpoint comprometido atraves do endpoint `PATCH /webhooks/:id` com `active = false`.

### [Risco 3] Lentidao Severa de Endpoint de Cliente Consumindo Threads do Worker

- **Probabilidade:** Alta
- **Impacto:** Medio
- **Mitigacao:**
  - Configuracao de timeout estrito e nao negociavel de 10 segundos para toda requisicao de despacho do worker.
  - Utilizacao de pool de sockets HTTP com conexoes persistentes (Keep-Alive) isolado no processo do worker.
- **Plano de contingencia:** Marcar o evento como falho por timeout e agendar retry imediato com backoff, liberando a thread para os proximos clientes.

---

### 11. Integracao com o sistema existente

Esta secao detalha as interfaces tecnicas exatas e caminhos de arquivos reais da base de codigo que serao integrados para a implementacao da feature de webhooks:

1. **`src/modules/orders/order.service.ts`:**
   - **Ponto de Integracao:** Metodo `changeStatus(id: string, input: UpdateOrderStatusInput, userId: string)` (linhas 131 a 179).
   - **Acao:** Dentro da transacao relacional `this.prisma.$transaction(async (tx) => { ... })`, logo apos a atualizacao em `tx.order.update` e a insercao em `tx.orderStatusHistory.create`, invocar a funcao pura `publishWebhookEvent(tx, refreshedOrder, from, to)`. Se a gravacao da outbox falhar, toda a transacao de mudanca de status do pedido sofre rollback.

2. **`src/config/database.ts`:**
   - **Ponto de Integracao:** Configuracao e gerenciamento da instancia do `PrismaClient` (linhas 1 a 15).
   - **Acao:** O processo independente do worker (`src/worker.ts`) importa a configuracao de banco, mas instancia seu proprio pool de conexoes Prisma dedicado. Isso garante que a volumetria de polling a cada 2 segundos nao consuma as conexoes reservadas para atender as requisicoes HTTP da API web em `src/server.ts`.

3. **`src/middlewares/auth.middleware.ts`:**
   - **Ponto de Integracao:** Funcao middleware `requireRole(...roles: AuthUser['role'][])` (linhas 49 a 61).
   - **Acao:** Aplicada na rota administrativa de reprocessamento de DLQ (`POST /admin/webhooks/dead-letter/:id/replay`). O middleware intercepta a requisicao, valida se o `req.user.role === 'ADMIN'` e rejeita usuarios com papel `OPERATOR` com erro `ForbiddenError('Insufficient permissions')` (HTTP 403).

4. **`src/shared/errors/app-error.ts`:**
   - **Ponto de Integracao:** Classe base de erros tipados da aplicacao `AppError` (linhas 1 a 17).
   - **Acao:** Criacao do arquivo `src/modules/webhooks/webhook.errors.ts` com classes especializadas herdando de `AppError` (`WebhookNotFoundError`, `WebhookInvalidUrlError`, `WebhookInsecureUrlError`, `WebhookPayloadTooLargeError`), padronizando o codigo de erro com prefixo `WEBHOOK_*` e status HTTP adequado.

5. **`src/shared/logger/index.ts`:**
   - **Ponto de Integracao:** Utilitario singleton `logger` baseado na biblioteca Pino (linhas 1 a 20).
   - **Acao:** Utilizacao do logger em `src/worker.ts` e no controller de webhooks para saida padronizada em formato JSON estruturado, registrando metricas de duracao de despacho (`duration_ms`), identificadores unicos (`event_id`, `webhook_id`) e diagnostico detalhado de falhas de rede.

6. **`src/middlewares/error.middleware.ts`:**
   - **Ponto de Integracao:** Interceptador global de excecoes da aplicacao Express `errorMiddleware` (linhas 1 a 66).
   - **Acao:** Processa automaticamente qualquer erro do modulo de webhooks que herde de `AppError` ou seja gerado pelo Zod/Prisma, serializando a resposta de erro padronizada `{ error: { code, message, details } }` sem necessidade de customizacao no middleware central.
