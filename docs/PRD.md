### PRD: Order Management System (OMS) Sistema de Webhooks de Notificação de Pedidos

Versão: 1.0.0
Data: 2026-09-15
Responsável: Marcos (Product Manager)

---

### Resumo

O Order Management System (OMS) gerencia o ciclo de vida comercial dos pedidos da plataforma, controlando desde a criação e reserva de estoque até a expedição e entrega final. Atualmente, os clientes corporativos B2B necessitam acompanhar ativamente as mudanças de status de seus pedidos, mas a plataforma não disponibiliza nenhum canal reativo para emissão de notificações assíncronas. Para obter as atualizações, os clientes realizam consultas repetitivas de varredura (*polling*) no endpoint `GET /orders`, sobrecarregando o banco de dados e a infraestrutura da API.

Esta feature implementa uma esteira automatizada de notificações ativas de saída (*outbound webhooks*). A cada transição de estado no pedido (ex: `PAID`, `PROCESSING`, `SHIPPED`, `DELIVERED`, `CANCELLED`), o sistema registra o evento atomicamente na tabela outbox do MySQL e um worker desacoplado dispara uma requisição HTTP POST segura em tempo real (latência ponta a ponta inferior a 10 segundos) para o endpoint cadastrado pelo cliente, garantindo entrega confiável (*at-least-once*), integridade com assinatura HMAC-SHA256 e resiliência via retentativas com backoff exponencial e Dead Letter Queue (DLQ).

---

### Contexto e problema

Público-alvo
- Desenvolvedores e Integradores B2B: Engenheiros de software e arquitetos dos parceiros comerciais que constroem integrações entre seus sistemas corporativos (ERPs, TMSs e WMSs) e o nosso OMS.
- Operadores Logísticos e Fiscais dos Clientes: Usuários de negócio que dependem da confirmação em tempo real de status (`SHIPPED`, `DELIVERED`) para emissão de notas fiscais e liberação de transporte.
- Administradores Internos da Plataforma: Equipe de suporte e sustentação que audita a entrega de notificações e opera o reprocessamento manual de eventos com falha via DLQ.

Cenários de uso chave
- Notificação de Separação em Tempo Real: Operador atualiza pedido para `PROCESSING` e o WMS da Atlas Comercial recebe o webhook assinado em menos de 3 segundos para alocar docas e etiquetas de expedição sem intervenção manual.
- Tolerância a Manutenção Programada do Cliente: Servidor receptor da MaxDistribuição fica offline por 2 horas durante manutenção; o motor de webhooks retém eventos e aplica retentativas progressivas com backoff exponencial (1m, 5m, 30m, 2h), entregando com sucesso após o retorno sem perda de integridade.
- Auditoria e Desduplicação Segura no Integrador: Oscilação de rede provoca retransmissão de notificação; o receptor da Nova Cargo verifica o cabeçalho `X-Event-Id` único, identifica duplicidade, descarta efeitos colaterais e responde `200 OK` de forma idempotente.

Onde essa feature será implantada
- Backend do Order Management System (OMS), acoplado à transação do serviço de pedidos (`src/modules/orders/services/order.service.ts`), com novo módulo REST de gerenciamento de webhooks (`src/modules/webhooks`) na API Express, persistência em tabelas relacionais do MySQL (`webhook_outbox`, `webhook_dead_letter`, etc.) gerenciadas via Prisma ORM, e execução de worker desacoplado em processo independente de background (`src/worker.ts`).

Problemas priorizados
- Ineficiência Operacional e Sobrecarga de Infraestrutura: Varredura constante via `GET /orders` sobrecarrega pool de conexões MySQL e gera picos artificiais de CPU na API pública de pedidos. Impacto: Alto. Prioridade: Alta.
- Risco Iminente de Churn de Clientes Estratégicos: Atlas Comercial, MaxDistribuição e Nova Cargo exigem notificações em tempo real, com ultimato formal da Atlas de migração para concorrente caso não seja entregue até o encerramento do trimestre (final de novembro de 2026). Impacto: Crítico. Prioridade: Alta.
- Inviabilidade de Chamadas Síncronas no Fluxo Transacional: Acoplar chamadas HTTP remotas dentro de `OrderService.changeStatus` bloquearia conexões locais de banco de dados e propagaria falhas em cascata para todos os usuários da aplicação. Impacto: Crítico. Prioridade: Alta.

---

### Objetivos e métricas

| Objetivo | Métrica | Meta |
| ---------------------------------------------------------------------- | --------------------------------------------------------------- | ------------------------- |
| Tempo de Resposta em Tempo Real | Latência ponta a ponta (P99) entre o commit da alteração de status e o recebimento pelo cliente | < 10 segundos |
| Eficácia de Despacho Inicial | Taxa de sucesso na primeira tentativa de envio de notificações | >= 95% |
| Redução de Carga na API | Redução do volume de chamadas de varredura contínua no endpoint `GET /orders` por clientes corporativos | >= 90% em até 30 dias após lançamento |
| Zero Perda de Eventos Commitados | Consistência atômica entre atualização do pedido e criação do evento (Zero Dual-Write) | 100% (Invariante relacional) |
| Retenção de Clientes Críticos | Evitar cancelamento de contrato (churn) de Atlas Comercial, MaxDistribuição e Nova Cargo | 0% churn atribuível à ausência de webhooks |
| Prazo de Entrega Contratual | Conclusão do desenvolvimento, homologação e deploy em produção | 3 Sprints (Até final de novembro de 2026) |

---

### Escopo

Incluso
- Cadastro, edição, listagem e inativação de endpoints de webhooks por cliente via API REST.
- Associação de lista de status de interesse por endpoint (filtro de eventos na origem).
- Persistência atômica do evento na tabela `webhook_outbox` acoplada à transação do pedido (`OrderService.changeStatus`).
- Despacho assíncrono via worker dedicado executando polling a cada 2 segundos.
- Autenticação e integridade de requisições via assinatura criptográfica HMAC-SHA256 no cabeçalho `X-Signature`.
- Chave secreta única por endpoint gerada automaticamente pela plataforma (32 bytes aleatórios).
- Mecanismo de rotação de secret com período de carência (*grace period*) de 24 horas.
- Cabeçalhos padronizados de segurança e controle: `X-Event-Id`, `X-Timestamp`, `X-Webhook-Id`.
- Garantia de entrega *at-least-once* com desduplicação por `X-Event-Id`.
- Política de retentativas automáticas em 5 etapas com backoff exponencial (1m, 5m, 30m, 2h, 12h) e full jitter aleatório.
- Segregação de eventos permanentemente falhos em tabela de Dead Letter Queue (`webhook_dead_letter`).
- Rota administrativa para consulta e reprocessamento manual (*replay*) de mensagens da DLQ com restrição à role `ADMIN`.
- Consulta ao histórico das últimas 100 entregas (`GET /webhooks/:id/deliveries`).
- Validação estrita de protocolo HTTPS obrigatório e limite máximo de payload de 64KB.

Fora de escopo
- Webhooks de entrada (*inbound webhooks*): recebimento e processamento de notificações enviadas por terceiros.
- Interface gráfica de usuário (painel/dashboard visual): gestão 100% via API REST; desenvolvimento de frontend visual tratado posteriormente.
- Alertas e notificações ativas por canais alternativos (e-mail ou SMS) comunicando falhas consecutivas de entrega.
- Controle de vazão de saída (*outbound rate limiting*) por cliente no lançamento inicial.
- Garantia de ordenação global irrestrita entre pedidos distintos (ordenação mantida estritamente por pedido via single-worker).
- Contratação, provisionamento ou operação de infraestrutura de mensageria dedicada (Apache Kafka, RabbitMQ, Redis Cluster).

---

### Requisitos funcionais

#### [RF-01] Cadastro de Endpoint de Webhook
A API deve disponibilizar endpoint `POST /webhooks` para registro de novos destinos de notificações, vinculando URL HTTPS, eventos inscritos e gerando secret criptográfica única para o cliente.

**Fluxo principal**
- O cliente autenticado envia requisição `POST /webhooks` contendo `url`, `events` (ex: `PAID`, `PROCESSING`, `SHIPPED`, `DELIVERED`, `CANCELLED`) e `customerId`.
- O sistema valida os campos via schema Zod, exigindo protocolo HTTPS e ao menos um evento válido.
- O sistema gera uma chave secreta aleatória de 32 bytes (codificada em hexadecimal).
- O sistema persiste o registro na tabela de webhooks com status `ACTIVE`.
- O sistema retorna HTTP 201 Created com os dados do webhook e a chave `secret` em texto claro.

**Fluxos alternativos e exceções**
- Cadastro de múltiplos webhooks para o mesmo cliente: permitido para URLs distintas ou com filtros de eventos complementares.

**Erros previstos**
- 400 Bad Request: URL insegura (`http://`), URL malformada, formato inválido ou array de eventos vazio/inválido.
- 401 Unauthorized: Token de autenticação ausente ou inválido.
- 403 Forbidden: Usuário sem permissão para cadastrar recursos para o `customerId` informado.

**Prioridade:** alta

---

#### [RF-02] Listagem de Endpoints por Cliente
A API deve disponibilizar endpoint `GET /webhooks` para consultar e listar os webhooks cadastrados por um cliente corporativo.

**Fluxo principal**
- O cliente autenticado envia requisição `GET /webhooks?customerId=:id` com parâmetros de paginação opcionais (`page`, `limit`).
- O sistema valida a permissão de acesso ao `customerId`.
- O sistema consulta os webhooks cadastrados vinculados ao cliente no banco de dados.
- O sistema retorna HTTP 200 OK contendo a lista de webhooks com a chave `secret` devidamente mascarada (ex: `sec_****...abcd`).

**Fluxos alternativos e exceções**
- Cliente sem webhooks cadastrados: o sistema retorna HTTP 200 OK com lista vazia `[]`.

**Erros previstos**
- 400 Bad Request: Parâmetros de paginação inválidos ou `customerId` malformado.
- 401 Unauthorized: Token de autenticação ausente ou inválido.
- 403 Forbidden: Tentativa de listar webhooks pertencentes a outro cliente.

**Prioridade:** media

---

#### [RF-03] Atualização de Configuração de Webhook
A API deve disponibilizar endpoint `PATCH /webhooks/:id` para atualizar parcialmente a URL de destino, a lista de eventos de interesse ou ativar/desativar o endpoint.

**Fluxo principal**
- O cliente autenticado envia requisição `PATCH /webhooks/:id` contendo os campos a serem alterados (`url`, `events`, `status`).
- O sistema valida os novos valores via Zod (assegurando protocolo HTTPS se `url` for informada).
- O sistema atualiza o registro no banco de dados e atualiza o timestamp `updated_at`.
- O sistema retorna HTTP 200 OK com os dados atualizados e secret mascarada.

**Fluxos alternativos e exceções**
- Desativação do endpoint (`status: INACTIVE`): novos eventos deixam de ser agendados para este webhook; disparos já em processamento finalizam normalmente.

**Erros previstos**
- 400 Bad Request: Payload inválido, URL insegura (`http://`) ou evento desconhecido.
- 401 Unauthorized: Token de autenticação ausente ou inválido.
- 403 Forbidden: Usuário sem privilégio para alterar o recurso.
- 404 Not Found: Webhook não encontrado ou não pertencente ao cliente autenticado.

**Prioridade:** alta

---

#### [RF-04] Remoção de Webhook
A API deve disponibilizar endpoint `DELETE /webhooks/:id` para exclusão de um endpoint cadastrado.

**Fluxo principal**
- O cliente autenticado envia requisição `DELETE /webhooks/:id`.
- O sistema localiza o webhook e valida a titularidade do recurso.
- O sistema remove o registro ou aplica exclusão lógica, cessando novos agendamentos.
- O sistema retorna HTTP 204 No Content.

**Fluxos alternativos e exceções**
- Eventos pendentes na outbox vinculados ao webhook removido são cancelados durante a leitura do worker.

**Erros previstos**
- 401 Unauthorized: Token de autenticação ausente ou inválido.
- 403 Forbidden: Usuário sem privilégio para excluir o recurso.
- 404 Not Found: Webhook com o ID especificado não existe ou pertence a outro cliente.

**Prioridade:** media

---

#### [RF-05] Filtragem de Eventos na Origem
O sistema deve verificar na transição de status do pedido se existem webhooks ativos cadastrados para o cliente e status de destino antes de gravar na outbox.

**Fluxo principal**
- O método `OrderService.changeStatus` é executado para alterar o estado de um pedido.
- O sistema consulta se existem webhooks com status `ACTIVE` cadastrados para aquele `customerId` contendo o novo status em sua lista de eventos.
- Se houver ao menos um webhook ativo inscrito, o sistema gera os registros de evento para persistência na tabela outbox.

**Fluxos alternativos e exceções**
- Se nenhum webhook ativo estiver inscrito no status de destino, nenhuma linha é gravada na tabela outbox, concluindo a transação sem I/O adicional.

**Erros previstos**
- Falha de consulta no banco de dados durante a verificação: aborta a transação do pedido (rollback) para preservar integridade.

**Prioridade:** alta

---

#### [RF-06] Persistência Atômica do Snapshot na Outbox
Ao ocorrer alteração de status do pedido (`changeStatus`), o sistema deve gravar uma linha na tabela `webhook_outbox` dentro da mesma transação SQL relacional do pedido.

**Fluxo principal**
- Durante a execução de `OrderService.changeStatus`, o método `publishWebhookEvent` é executado recebendo a transação SQL ativa (`tx`).
- O sistema gera um UUID v4 imutável (`eventId`) e monta o snapshot completo do pedido com timestamp e metadados.
- O sistema valida que o tamanho do payload serializado em JSON não excede 64KB.
- O sistema insere o registro na tabela `webhook_outbox` com status `PENDING`, `attempts = 0` e `next_retry_at = NOW()`.
- A transação SQL do pedido é commitada atomicamente.

**Fluxos alternativos e exceções**
- Se a transação do pedido sofrer rollback por falha de estoque ou regra de negócio, a gravação na outbox é revertida simultaneamente (zero dual-write).

**Erros previstos**
- 400 Bad Request / Exceção de validação: Tamanho do payload superior a 64KB.
- 500 Internal Server Error: Falha de conexão ou concorrência relacional no MySQL.

**Prioridade:** alta

---

#### [RF-07] Despacho Assíncrono com Polling de 2 Segundos
Um worker desacoplado em processo independente deve consultar a tabela outbox a cada 2 segundos, ler lotes pendentes e disparar requisições HTTP para os destinos.

**Fluxo principal**
- A cada 2 segundos, o worker consulta a tabela `webhook_outbox` buscando até 50 eventos com `status IN ('PENDING', 'RETRY')` e `next_retry_at <= NOW()`, ordenados por `created_at ASC`.
- O worker altera o status dos eventos selecionados para `PROCESSING`.
- O worker recupera a URL de destino e a `secret` ativa do endpoint.
- O worker assina a mensagem, injeta os cabeçalhos obrigatórios e envia requisição HTTP POST com timeout de 10 segundos.
- Ao receber resposta HTTP 2xx (200, 201, 204), o worker marca o evento como `DELIVERED` e grava o registro na tabela de entregas.

**Fluxos alternativos e exceções**
- Se o servidor do cliente responder com código diferente de 2xx, timeout de 10s ou erro de socket, o evento entra no fluxo de retentativa automática (RF-08).
- Se o endpoint de destino tiver sido desativado ou removido, o evento é marcado como cancelado.

**Erros previstos**
- Falha temporária de conexão com o banco MySQL: o worker registra log de erro e aguarda o próximo ciclo de 2 segundos.
- Timeout de conexão ou handshake TLS com o servidor do cliente: aciona política de retry.

**Prioridade:** alta

---

#### [RF-08] Retentativas Automáticas com Backoff Exponencial e DLQ
O sistema deve executar até 5 retentativas com backoff exponencial e jitter em caso de falha de envio; esgotadas as tentativas, deve transferir o evento para a Dead Letter Queue.

**Fluxo principal**
- Após falha no disparo de um evento, o worker incrementa o campo `attempts`.
- Se `attempts < 5`, o worker calcula o próximo agendamento utilizando a escala de backoff exponencial (1m, 5m, 30m, 2h, 12h) acrescida de full jitter aleatório (+-20%).
- O worker atualiza o registro na `webhook_outbox` para status `RETRY` com o novo `next_retry_at`.
- O evento permanece retido até o próximo ciclo elegível de polling.

**Fluxos alternativos e exceções**
- Se `attempts >= 5` e a tentativa falhar, o worker move o registro da tabela `webhook_outbox` para a tabela `webhook_dead_letter`, gravando os dados do evento, o histórico dos erros e marcando status definitivo `DEAD_LETTER`.

**Erros previstos**
- Falha de escrita no MySQL ao mover evento para a tabela de DLQ: o evento permanece retido com status de erro para tratamento sem perda de dados.

**Prioridade:** alta

---

#### [RF-09] Consulta ao Histórico de Entregas
A API deve disponibilizar endpoint `GET /webhooks/:id/deliveries` para que os clientes corporativos consultem as últimas 100 tentativas de envio de suas notificações.

**Fluxo principal**
- O cliente autenticado envia requisição `GET /webhooks/:id/deliveries?limit=50`.
- O sistema valida a titularidade do webhook para o cliente solicitante.
- O sistema consulta a tabela de histórico de entregas vinculada ao webhook, retornando `eventId`, `statusCode`, `executionTimeMs`, `attemptNumber`, `deliveredAt` e eventuais mensagens de erro.
- O sistema retorna HTTP 200 OK com a lista ordenada das tentativas de entrega.

**Fluxos alternativos e exceções**
- Webhook recém-cadastrado sem envios: retorna HTTP 200 OK com lista vazia `[]`.

**Erros previstos**
- 401 Unauthorized: Token de autenticação ausente ou inválido.
- 403 Forbidden: Usuário sem autorização para o webhook informado.
- 404 Not Found: Webhook não encontrado.

**Prioridade:** media

---

#### [RF-10] Rotação Segura de Credencial (Secret)
A API deve disponibilizar endpoint `POST /webhooks/:id/rotate-secret` para gerar nova chave criptográfica mantendo a chave anterior válida por período de carência de 24 horas.

**Fluxo principal**
- O cliente autenticado envia requisição `POST /webhooks/:id/rotate-secret`.
- O sistema gera uma nova secret de 32 bytes aleatórios.
- O sistema armazena a secret anterior em `previous_secret` com expiração definida para `NOW() + 24 horas` e define a nova chave como `secret` ativa.
- O sistema retorna HTTP 200 OK contendo a nova chave secreta e a data/hora limite de expiração da chave anterior.

**Fluxos alternativos e exceções**
- Nova solicitação de rotação dentro da janela de 24 horas: a chave intermediária é invalidada e substituída pela nova chave gerada.

**Erros previstos**
- 401 Unauthorized: Token de autenticação ausente ou inválido.
- 403 Forbidden: Usuário sem privilégio para rotacionar credenciais.
- 404 Not Found: Webhook não encontrado.

**Prioridade:** media

---

#### [RF-11] Replay Administrativo de Eventos da DLQ
A API deve disponibilizar endpoint `POST /admin/webhooks/dead-letter/:id/replay`, restrito a usuários com perfil `ADMIN`, para reenfileirar eventos falhos da DLQ na outbox.

**Fluxo principal**
- O administrador autenticado envia requisição `POST /admin/webhooks/dead-letter/:id/replay`.
- O sistema valida a autenticidade do token JWT e verifica a presença da role `ADMIN`.
- O sistema localiza o registro na tabela `webhook_dead_letter`.
- O sistema insere um novo registro correspondente na tabela `webhook_outbox` com status `PENDING`, `attempts = 0` e `next_retry_at = NOW()`.
- O sistema registra evento de auditoria contendo ID do administrador, timestamp e justificativa.
- O sistema retorna HTTP 200 OK confirmando o reenfileiramento do evento.

**Fluxos alternativos e exceções**
- Usuário autenticado sem role `ADMIN` (ex: `OPERATOR` ou usuário comum): o sistema rejeita imediatamente o acesso.

**Erros previstos**
- 401 Unauthorized: Token ausente ou inválido.
- 403 Forbidden: Usuário não possui perfil `ADMIN`.
- 404 Not Found: Evento de DLQ com o ID informado não existe.

**Prioridade:** media

---

#### [RF-12] Assinatura Digital e Cabeçalhos Padronizados
O worker deve assinar digitalmente todas as requisições HTTP de saída com HMAC-SHA256 e injetar cabeçalhos padronizados de rastreamento e desduplicação.

**Fluxo principal**
- Antes de disparar a requisição HTTP POST, o worker obtém o payload JSON do evento, a `secret` ativa do endpoint e o timestamp Unix atual.
- O worker computa o hash criptográfico HMAC-SHA256 utilizando a `secret` como chave simétrica e o corpo serializado como dado.
- O worker injeta os cabeçalhos obrigatórios na requisição HTTP:
  - `X-Event-Id`: identificador UUID v4 único para desduplicação idempotente;
  - `X-Signature`: assinatura HMAC-SHA256 em codificação hexadecimal;
  - `X-Timestamp`: epoch Unix do instante de envio para validação contra replay attacks;
  - `X-Webhook-Id`: identificador cadastral do webhook de destino;
  - `Content-Type`: `application/json`.
- O worker envia a requisição HTTP POST para o endpoint do cliente.

**Fluxos alternativos e exceções**
- Endpoints em período de carência de rotação: o receptor valida a assinatura com a nova chave ou com a anterior até a expiração de 24h.

**Erros previstos**
- Erro interno de serialização ou chave ausente: o worker loga erro com severidade ERROR e suspende a tentativa para evitar disparos malformados.

**Prioridade:** alta

---

### Requisitos não funcionais

Performance
- Latência ponta a ponta P99 inferior a 10 segundos entre o commit da transação do pedido e o recebimento pelo cliente.
- Polling do worker configurado em ciclos de 2 segundos com leitura em lotes de até 50 eventos para não sobrecarregar I/O de banco.
- Timeout estrito de 10 segundos em todas as chamadas HTTP externas aos servidores dos clientes.
- Limite máximo de payload de 64KB por evento JSON para evitar consumo excessivo de memória e lentidão de rede.

Disponibilidade
- Uptime de 99,9% mensal em produção para a API de gerenciamento de webhooks e worker assíncrono.
- Resiliência operacional com janela de retentativas de até 15 horas (escala 1m, 5m, 30m, 2h, 12h) para absorver manutenções e indisponibilidades temporárias de parceiros.

Segurança e autorização
- Exigência mandatória de protocolo TLS/HTTPS para todos os endpoints cadastrados (rejeição de URLs `http://`).
- Assinatura digital HMAC-SHA256 em 100% dos payloads enviados via cabeçalho `X-Signature`, com secrets individuais de 32 bytes geradas com alta entropia.
- Prevenção contra ataques de repetição (*replay attacks*) via cabeçalho `X-Timestamp` com janela de tolerância recomendada de 5 minutos no receptor.
- Mecanismo de rotação de credenciais com período de convivência (*grace period*) de 24 horas.
- Autenticação obrigatória via JWT e autorização RBAC nas rotas da API, com restrição estrita do endpoint de replay da DLQ à role `ADMIN`.
- Mascaramento mandatório de chaves secretas em todas as saídas de APIs de consulta e logs estruturados.

Observabilidade
- Logs estruturados em formato JSON utilizando o logger Pino existente, com campos de correlação (`eventId`, `webhookId`, `customerId`, `attemptNumber`), sem exposição de chaves secretas.
- Métricas operacionais detalhadas por endpoint: taxas de entrega com sucesso, contagem de retentativas, latência de resposta HTTP e contagem de eventos na DLQ.
- Trilha de auditoria das tentativas de envio gravada no banco de dados com código HTTP retornado, tempo de resposta e detalhes de falhas.

Confiabilidade e integridade de dados
- Consistência transacional atômica via Padrão Outbox no MySQL: inserção do evento na tabela `webhook_outbox` executada dentro da mesma transação SQL de `OrderService.changeStatus` (zero dual-write e zero perda de eventos confirmados).
- Isolamento total de falhas externas: indisponibilidade, falhas ou lentidão nos clientes B2B não impactam a performance nem causam rollback na API de pedidos do OMS.
- Garantia de entrega *at-least-once* combinada com cabeçalho `X-Event-Id` único (UUID v4) para viabilizar desduplicação idempotente nos sistemas receptores.

Compatibilidade e portabilidade
- API RESTful seguindo padrões HTTP e respostas em JSON (`application/json`).
- Aplicação e worker desenvolvidos em Node.js 20 LTS e Express 4.x, empacotados em contêineres compatíveis com OCI (Docker).
- Mapeamento objeto-relacional e migrações declarativas utilizando Prisma ORM 5.x sobre banco MySQL 8.x.

Compliance
- Trilha de auditoria das últimas 100 tentativas de entrega (`GET /webhooks/:id/deliveries`) e histórico completo da tabela DLQ para auditorias fiscais e contratuais com parceiros B2B.
- Registro de auditoria contendo ID de usuário administrador, justificativa e timestamp para todas as ações de replay manual de eventos da DLQ.

Acessibilidade no frontend consumidor
- Não aplicável diretamente ao backend de webhooks (comunicação M2M assíncrona); contratos de erro da API seguem respostas padronizadas em JSON com mensagens claras e semânticas em português/inglês para facilitar integração futura com painel web administrativo acessível (WCAG 2.1 AA).

---

### Arquitetura e abordagem

Abordagem
- Adoção do padrão arquitetural Transacional Outbox acoplado ao ciclo de vida de pedidos existente no MySQL, complementado por um worker assíncrono desacoplado em processo independente (`src/worker.ts`) que consome os eventos pendentes via polling a cada 2 segundos, garantindo entrega at-least-once, blindagem do fluxo transacional central contra indisponibilidade de terceiros e eliminação do problema da escrita dupla (dual-write).

Componentes
- API Backend (Node.js / Express / Prisma): Responsável pelos endpoints REST de gerenciamento de webhooks (`/webhooks`), consulta de histórico, rotação de secrets, rota administrativa de replay e execução da regra de negócio de pedidos (`OrderService.changeStatus`).
- Banco de Dados Relacional (MySQL 8.x): Fonte única de verdade, mantendo tabelas transacionais de pedidos (`orders`, `order_status_history`, `stock_quantity`), tabela outbox (`webhook_outbox`), tabela de DLQ (`webhook_dead_letter`) e configurações de endpoints (`webhooks`, `webhook_deliveries`).
- Worker de Webhooks (`src/worker.ts`): Processo em background independente que executa polling a cada 2 segundos, processa eventos em lotes de até 50 registros, calcula assinaturas criptográficas HMAC-SHA256, efetua disparos HTTP com timeout de 10s e orquestra a política de retentativas e transição para DLQ.

Integrações
- OrderService (`changeStatus`): Durante a transição de estado de pedidos, invoca atomicamente a função `publishWebhookEvent` passando a transação SQL ativa para gravar o snapshot JSON na outbox se houver endpoints inscritos.
- Servidores Receptores de Clientes B2B (Atlas, MaxDistribuição, Nova Cargo): Receptores externos HTTPS que recebem as requisições HTTP POST com payload JSON e cabeçalhos de segurança (`X-Event-Id`, `X-Signature`, `X-Timestamp`, `X-Webhook-Id`), respondendo com códigos HTTP 2xx para confirmar recebimento.
- Módulo de Auditoria e DLQ: Interliga os eventos exauridos (após 5 tentativas falhas) à tabela `webhook_dead_letter` e expõe a rota administrativa para replay manual pelo time de operações/suporte.

### Decisões e trade-offs

#### Decisão: Padrão Transacional Outbox no MySQL vs Disparo HTTP Síncrono
- **Justificativa:** Garante consistência transacional ACID estrita (zero dual-write), eliminando riscos de falhas externas ou lentidão de terceiros travarem a transação de pedidos ou forçarem rollbacks indevidos no banco de dados.
- **Trade-off:** Assume-se a complexidade de manter uma tabela de eventos intermediária e um worker de polling em processo separado em troca de estabilidade e desacoplamento do núcleo transacional.

#### Decisão: Worker em Polling de 2 Segundos vs Broker Dedicado de Mensageria (Kafka / Redis Streams)
- **Justificativa:** Permite reaproveitar 100% da infraestrutura existente (MySQL/Node.js), reduzindo complexidade operacional, dispensando custos adicionais de servidores e viabilizando a entrega dentro do prazo contratual crítico de 3 sprints.
- **Trade-off:** Introduz uma latência basal de leitura de 0 a 2 segundos (perfeitamente tolerável frente à meta de latência < 10s) e gera uma carga leve e periódica de leitura no MySQL (mitigada por índices otimizados e limites por lote).

#### Decisão: Garantia At-Least-Once com Desduplicação por X-Event-Id vs Exactly-Once Distribuído
- **Justificativa:** Elimina a necessidade de protocolos distribuídos de coordenação complexos e de alto overhead (como Two-Phase Commit), permitindo resiliência simples e tolerância a falhas transitórias de rede.
- **Trade-off:** Transfere aos integradores parceiros a responsabilidade de implementar recepção idempotente e descartar notificações repetidas com base no cabeçalho `X-Event-Id`.

#### Decisão: Autenticação com HMAC-SHA256 e Secret Única com Grace Period vs mTLS / OAuth2
- **Justificativa:** Proporciona garantia de integridade e autenticidade ponta a ponta com implementação simples e amplamente suportada pelos integradores B2B, enquanto o grace period de 24 horas garante rotação de credenciais com zero downtime.
- **Trade-off:** Exige que os clientes B2B implementem a verificação criptográfica do hash HMAC nas suas aplicações receptoras.

---

### Dependências

#### Infraestrutura: Banco de Dados MySQL 8.x
Banco de dados relacional gerenciado já provisionado em ambiente de produção, com suporte nativo a tipos JSON e transações ACID necessários para a tabela outbox e controle de integridade relacional.

#### Framework e ORM: Node.js 20 LTS, Express 4.x e Prisma ORM 5.x
Ambiente de runtime da aplicação, framework HTTP e ORM já homologados na codebase, necessários para modelagem declarativa das migrações (`prisma/migrations`) e validação de esquemas com Zod.

#### Segurança: Auditoria de Segurança da Informação (SecOps)
Janela mandatória de homologação de 2 dias úteis reservada para auditoria do código de criptografia HMAC, geração de secrets com alta entropia e validação do mascaramento de dados sensíveis antes do deploy em produção.

#### Integração: Clientes Parceiros B2B (Atlas Comercial, MaxDistribuição e Nova Cargo)
Disponibilização de endpoints receptores em ambiente de homologação e validação conjunta das assinaturas HMAC, cabeçalhos de segurança e tolerância a retentativas durante a fase de testes integrados.

#### Observabilidade: Biblioteca Pino Logger
Infraestrutura interna de logs estruturados em JSON já padronizada no repositório para emissão de telemetria e rastreamento das etapas do worker sem expor credenciais sensíveis.

---

### Riscos e mitigação

#### Sobrecarga e contenção de concorrência no banco de dados MySQL devido ao polling contínuo do worker
- **Probabilidade:** media
- **Impacto:** alto
- **Mitigação:**
  - Criação de índice composto otimizado cobrindo exatamente o predicado de busca do worker: `INDEX idx_outbox_polling (status, next_retry_at, created_at)`.
  - Limitação rigorosa de leitura em pequenos lotes (`LIMIT 50`) com transações curtas para evitar bloqueios prolongados de linhas.
  - Implantação de rotina periódica de expurgo que remove registros entregues (`DELIVERED`) após 30 dias.
- **Plano de contingência:** Ajustar temporariamente o intervalo de polling do worker de 2s para 5s via variável de ambiente `WEBHOOK_POLLING_INTERVAL_MS` para reduzir a pressão imediata no banco até que recursos de CPU/memória da réplica ou banco sejam ampliados.

#### Vazamento de credenciais secretas por falha de armazenamento ou exposição acidental no lado do cliente B2B
- **Probabilidade:** media
- **Impacto:** alto
- **Mitigação:**
  - Geração de secrets com alta entropia criptográfica (32 bytes aleatórios).
  - Isolamento completo de credenciais, gerando secret estritamente individual por endpoint cadastrado.
  - Disponibilização de endpoint de rotação rápida de secret com período de carência de 24 horas, permitindo transição suave sem interrupção de serviço.
  - Mascaramento mandatório de todas as chaves secretas em logs, respostas de consulta de APIs e interfaces.
- **Plano de contingência:** Revogação imediata da credencial comprometida via API/banco pelo time de sustentação e geração de nova chave com contato direto com o time técnico do parceiro afetado.

#### Lentidão crônica ou esgotamento de recursos do worker provocado por instabilidade prolongada de servidores receptores
- **Probabilidade:** alta
- **Impacto:** medio
- **Mitigação:**
  - Aplicação de timeout rígido e não negociável de 10 segundos em todas as chamadas HTTP externas disparadas pelo worker.
  - Isolamento do worker em processo de background separado (`src/worker.ts`), impedindo que lentidões degradem o event loop da API principal do OMS.
  - Política de retentativas progressivas com backoff exponencial e full jitter, segregando eventos na DLQ após 5 tentativas falhas.
- **Plano de contingência:** Pausar temporariamente o endpoint problemático via flag `INACTIVE` na API de webhooks para suspender novos despachos para o receptor instável enquanto a equipe do cliente soluciona o incidente.

#### Evasão contratual (churn) da Atlas Comercial caso o prazo de entrega estipulado seja ultrapassado
- **Probabilidade:** media
- **Impacto:** alto
- **Mitigação:**
  - Escopo estritamente enxuto focado no backend, descartando desenvolvimento de interfaces visuais e brokers adicionais para assegurar entrega em 3 sprints.
  - Alinhamento semanal de demonstração e homologação técnica conjunta com o time de engenharia da Atlas Comercial.
- **Plano de contingência:** Disponibilização prévia de ambiente de staging/sandbox já na Sprint 2 para que a Atlas inicie a integração do WMS antes do lançamento final em produção.

---

### Critérios de aceitação
Checklist objetivo que define se a feature está pronta.

- [ ] O cliente corporativo consegue cadastrar um endpoint de webhook via `POST /webhooks` fornecendo URL HTTPS válida e lista de status desejados, recebendo a chave secreta gerada.
- [ ] Tentativas de cadastro com URLs inseguras (`http://`) ou dados malformados são rejeitadas com código HTTP 400 Bad Request.
- [ ] O cliente consegue consultar seus webhooks cadastrados via `GET /webhooks?customerId=:id` com as secrets devidamente mascaradas.
- [ ] A alteração de status de pedido em `OrderService.changeStatus` persiste o snapshot do evento na tabela `webhook_outbox` atomicamente; em caso de rollback na transação do pedido, nenhum evento é gerado.
- [ ] O worker desacoplado consome a tabela outbox a cada 2 segundos e entrega o evento no destino em menos de 10 segundos (P99 < 10s).
- [ ] Toda requisição HTTP enviada ao cliente possui os cabeçalhos obrigatórios `X-Event-Id`, `X-Signature`, `X-Timestamp`, `X-Webhook-Id` e `Content-Type: application/json`.
- [ ] A assinatura enviada em `X-Signature` é calculada com algoritmo HMAC-SHA256 sobre o corpo da mensagem e permite validação íntegra na ponta receptora.
- [ ] Endpoints receptores instáveis sofrem até 5 tentativas com backoff exponencial e jitter (1m, 5m, 30m, 2h, 12h).
- [ ] Eventos que falharem após a 5ª tentativa são transferidos para a tabela `webhook_dead_letter` com histórico da falha.
- [ ] O endpoint administrativo `POST /admin/webhooks/dead-letter/:id/replay` reenfileira o evento na outbox e restringe o acesso estritamente à role `ADMIN`.
- [ ] O endpoint `POST /webhooks/:id/rotate-secret` gera nova credencial e mantém a chave anterior válida por período de carência de 24 horas.
- [ ] O endpoint `GET /webhooks/:id/deliveries` retorna o histórico das últimas tentativas de envio com status code e tempo de execução.

---

### Testes e validação

Tipos de teste obrigatórios
- Testes unitários para regras críticas: Validação dos esquemas Zod (rejeição de URLs `http://`, validação de enums de eventos e formatos de payload), funções criptográficas de geração de secrets de 32 bytes e cálculo de assinatura HMAC-SHA256, e algoritmo de cálculo de backoff exponencial com jitter aleatório.
- Testes de integração para fluxo principal: Teste de atomicidade transacional comprovando que o rollback em `OrderService.changeStatus` aborta a inserção na `webhook_outbox`, teste de ciclo de polling do worker selecionando lotes e marcando status `DELIVERED`, teste da máquina de estados de retentativas migrando para `webhook_dead_letter`, e teste de controle de acesso RBAC no endpoint de replay da DLQ (permissão para `ADMIN`, bloqueio para `OPERATOR`).
- Testes de segurança e auditoria: Validação de mascaramento de chaves secretas em logs estruturados Pino e respostas de API, e auditoria de código conduzida pela equipe de segurança (SecOps) com foco na entropia de secrets e integridade de assinaturas.
- Testes ponta a ponta (E2E) com servidor mock: Simulação completa de ciclo de vida com servidor HTTP mock recebendo notificações em tempo real, validando headers (`X-Event-Id`, `X-Signature`, `X-Timestamp`), simulação de indisponibilidade com códigos HTTP 500/timeouts para validar as 5 retentativas e teste de desduplicação idempotente pelo cliente.

Estratégia de validação
- TDD para lógica crítica de cálculo criptográfico HMAC, esquemas Zod e algoritmo de backoff exponencial com jitter; execução de suíte de testes de integração automatizados em pipeline de CI/CD contra contêiner MySQL real; testes de carga em ambiente de homologação para validação da latência P99 < 10s; e validação assistida com os times técnicos de Atlas Comercial, MaxDistribuição e Nova Cargo em ambiente de staging.
