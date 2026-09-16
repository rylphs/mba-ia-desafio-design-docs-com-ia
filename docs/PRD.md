# PRD — Sistema de Webhooks de Notificação de Pedidos

| Metadado | Detalhe |
| :--- | :--- |
| **Produto / Sistema** | Order Management System (OMS) |
| **Feature** | Sistema de Webhooks de Notificação de Pedidos (*Outbound Webhooks*) |
| **Product Manager (PM)** | Marcos |
| **Tech Lead** | Larissa |
| **Equipe de Engenharia** | Time de Pedidos & Time de Plataforma |
| **Status** | Aprovado |
| **Data de Criação** | 2026-09-13 |
| **Última Atualização** | 2026-09-15 |
| **Versão** | 1.0.0 |

---

## 1. Resumo e Contexto da Feature

O Order Management System (OMS) gerencia o ciclo de vida comercial dos pedidos da plataforma, controlando desde a criação e reserva de estoque até a expedição e entrega final. Atualmente, os clientes corporativos B2B necessitam acompanhar ativamente as mudanças de status de seus pedidos, mas a plataforma não disponibiliza nenhum canal reativo para emissão de notificações assíncronas.

Para obter as atualizações, os clientes realizam consultas repetitivas de varredura (*polling*) no endpoint `GET /orders`. Esse modelo gera consumo ineficiente de recursos de infraestrutura (CPU, conexões de rede e banco de dados) e degrada a experiência dos integradores. 

Esta feature implementa uma esteira automatizada de notificações ativas de saída (*outbound webhooks*). A cada transição de estado no pedido (ex: `PAID`, `PROCESSING`, `SHIPPED`, `DELIVERED`, `CANCELLED`), o sistema dispara uma requisição HTTP POST segura em tempo real para o endpoint cadastrado pelo cliente, garantindo entrega confiável com latência ponta a ponta inferior a 10 segundos.

---

## 2. Problema e Motivação

### 2.1 Ineficiência Operacional e Sobrecarga de Infraestrutura
Grandes clientes corporativos mantêm rotinas automatizadas que consultam o endpoint `GET /orders` em intervalos curtos para verificar alterações de estado. Essa varredura constante:
- Sobrecarrega o pool de conexões com o banco de dados MySQL;
- Gera picos artificiais de processamento de CPU na API pública de pedidos;
- Eleva os custos operacionais de tráfego e infraestrutura de rede.

### 2.2 Risco Iminente de Evasão Contratual (*Churn*) de Clientes Estratégicos
Três grandes clientes responsáveis por fração substancial do faturamento B2B da plataforma formalizaram a necessidade de notificações em tempo real:
- **Atlas Comercial**
- **MaxDistribuição**
- **Nova Cargo**

A **Atlas Comercial** comunicou formalmente à gerência de produto que, caso a funcionalidade de notificações em tempo real não seja entregue até o encerramento do trimestre corrente (final de novembro de 2026), iniciará a migração de suas operações logísticas para uma plataforma concorrente.

### 2.3 Inviabilidade do Modelo Síncrono
O método central de transição de status (`OrderService.changeStatus`) executa uma transação atômica que atualiza tabelas críticas (`orders`, `order_status_history` e `stock_quantity`). Não é aceitável acoplar chamadas HTTP remotas diretamente dentro dessa transação, pois a lentidão ou instabilidade de terceiros bloquearia conexões locais de banco de dados, propagando falhas em cascata para todos os usuários da aplicação.

---

## 3. Público-Alvo e Cenários de Uso

### 3.1 Público-Alvo
- **Desenvolvedores e Integradores B2B:** Engenheiros de software e arquitetos dos parceiros comerciais que constroem integrações entre seus sistemas corporativos (ERPs, TMSs e WMSs) e o nosso OMS.
- **Operadores Logísticos e Fiscais dos Clientes:** Usuários de negócio que dependem da confirmação em tempo real de expedição (`SHIPPED`) ou entrega (`DELIVERED`) para emissão de notas fiscais e liberação de transporte.
- **Administradores Internos da Plataforma:** Equipe de suporte e sustentação que audita a entrega de notificações e opera o reprocessamento manual de eventos com falha.

### 3.2 Cenários de Uso

#### Cenário 1: Notificação de Separação de Pedido em Tempo Real
- **Ator:** Sistema de WMS da Atlas Comercial.
- **Fluxo:** Um operador do OMS atualiza o pedido para `PROCESSING`. Automaticamente, a esteira de webhooks captura o evento e despacha uma notificação assinada via HMAC-SHA256 para o servidor da Atlas em menos de 3 segundos. O sistema da Atlas aloca as docas e etiquetas de expedição sem intervenção manual.

#### Cenário 2: Tolerância a Manutenção Programada do Cliente
- **Ator:** Servidor receptor da MaxDistribuição em manutenção de 2 horas.
- **Fluxo:** O OMS altera múltiplos pedidos para `SHIPPED`. O receptor da MaxDistribuição recusa as conexões. O motor de webhooks retém os eventos e aplica a política de retentativas progressivas com backoff exponencial e jitter (1m, 5m, 30m, 2h). Quando os servidores da MaxDistribuição retornam, recebem todas as notificações pendentes sem perda de integridade.

#### Cenário 3: Auditoria e Desduplicação Segura no Integrador
- **Ator:** Integrador técnico da Nova Cargo.
- **Fluxo:** Devido a uma oscilação de socket na confirmação HTTP, o worker do OMS retransmite uma notificação. O receptor da Nova Cargo verifica o cabeçalho `X-Event-Id` único e identifica que a mensagem já havia sido processada, descartando o efeito colateral duplicado e retornando `200 OK` de forma idempotente.

---

## 4. Objetivos e Métricas de Sucesso

| Objetivo | Métrica / Indicador | Meta Quantitativa |
| :--- | :--- | :--- |
| **Tempo de Resposta em Tempo Real** | Latência ponta a ponta ($P99$) entre o commit da alteração de status e o recebimento pelo cliente | **$< 10\text{ segundos}$** |
| **Eficácia de Despacho Inicial** | Taxa de sucesso na primeira tentativa de envio de notificações | **$\ge 95\%$** |
| **Redução de Carga na API** | Redução do volume de chamadas de varredura contínua no endpoint `GET /orders` por clientes corporativos | **$\ge 90\%$** em 30 dias após lançamento |
| **Zero Perda de Eventos Commitados** | Consistência atômica entre atualização do pedido e criação do evento (Zero Dual-Write) | **$100\%$** (Invariante relacional) |
| **Retenção de Clientes Críticos** | Evitar cancelamento de contrato (*churn*) da Atlas Comercial, MaxDistribuição e Nova Cargo | **$0\%$ churn** atribuível à ausência de webhooks |
| **Prazo de Entrega Contratual** | Conclusão do desenvolvimento, homologação e deploy em produção | **3 Sprints** (Até final de novembro de 2026) |

---

## 5. Escopo

### 5.1 Incluso no Escopo
- Cadastro, edição, listagem e inativação de endpoints de webhooks por cliente via API REST.
- Associação de lista de status de interesse por endpoint (filtro de eventos).
- Persistência atômica do evento na tabela `webhook_outbox` acoplada à transação do pedido (`OrderService.changeStatus`).
- Despacho assíncrono via worker dedicado executando polling a cada 2 segundos.
- Autenticação e integridade de requisições via assinatura criptográfica HMAC-SHA256 no cabeçalho `X-Signature`.
- Chave secreta única por endpoint gerada automaticamente pela plataforma.
- Mecanismo de rotação de secret com período de carência (*grace period*) de 24 horas.
- Cabeçalhos de segurança e controle: `X-Event-Id`, `X-Timestamp`, `X-Webhook-Id`.
- Garantia de entrega *at-least-once* com desduplicação por `X-Event-Id`.
- Política de retentativas automáticas em 5 etapas com backoff exponencial (1m, 5m, 30m, 2h, 12h) e full jitter.
- Segregação de eventos permanentemente falhos em tabela de Dead Letter Queue (`webhook_dead_letter`).
- Rota administrativa para consulta e reprocessamento manual (*replay*) de mensagens da DLQ com restrição à role `ADMIN`.
- Consulta ao histórico das últimas 100 entregas (`GET /webhooks/:id/deliveries`).
- Validação estrita de protocolo HTTPS obrigatório e limite máximo de payload de 64KB.

### 5.2 Fora de Escopo (Exclusões Explícitas)
1. **Webhooks de Entrada (*Inbound Webhooks*):** O sistema não receberá nem processará notificações enviadas por terceiros; a solução cobre exclusivamente o envio da plataforma para os clientes.
2. **Interface Gráfica de Usuário (Painel / Dashboard Visual):** A gestão dos webhooks será realizada 100% via endpoints de API RESTful. O desenvolvimento de painéis administrativos visuais será tratado posteriormente pelo time de frontend.
3. **Alertas e Notificações Ativas por Canais Alternativos (E-mail / SMS):** O sistema não enviará e-mails automáticos comunicando falhas consecutivas de entrega aos clientes neste primeiro release.
4. **Controle de Vazão de Saída (*Outbound Rate Limiting*):** Não será aplicada limitação de taxa artificial por cliente no envio de webhooks; o volume real de tráfego será monitorado em produção para avaliação futura.
5. **Garantia de Ordenação Global Irrestrita:** A ordenação é mantida estritamente por pedido (`order_id`) sob modelo de single-worker, sem garantia de sequenciamento global entre pedidos de clientes distintos.
6. **Infraestrutura de Mensageria Dedicada:** Não faz parte do escopo a contratação, provisionamento ou operação de ferramentas externas como Apache Kafka, RabbitMQ ou clusters Redis.

---

## 6. Requisitos Funcionais

- **[RF-01] Cadastro de Endpoint de Webhook:** A API deve disponibilizar endpoint `POST /webhooks` para registro de novos destinos, recebendo URL HTTPS, lista de status inscritos e `customerId`. O sistema gera e retorna a `secret` criptográfica correspondente.
- **[RF-02] Listagem de Endpoints por Cliente:** A API deve permitir a consulta paginada dos webhooks vinculados a um cliente através de `GET /webhooks?customerId=:id`, retornando os metadados cadastrados com a chave secreta devidamente mascarada.
- **[RF-03] Atualização de Configuração:** A API deve permitir alteração da URL de destino, da lista de eventos de interesse e do status ativo/inativo do webhook através de `PATCH /webhooks/:id`.
- **[RF-04] Remoção de Webhook:** A API deve permitir a exclusão de endpoints cadastrados via `DELETE /webhooks/:id`.
- **[RF-05] Filtragem de Eventos na Origem:** O sistema deve verificar, no momento da transição de status do pedido, se o cliente possui webhooks ativos configurados para o status de destino. Caso nenhum webhook tenha interesse no evento, nenhuma linha deve ser inserida na outbox.
- **[RF-06] Persistência Atômica do Snapshot na Outbox:** Ao ocorrer a alteração de status (`changeStatus`), o sistema deve gravar uma linha na tabela `webhook_outbox` dentro da mesma transação SQL relacional do pedido, serializando um snapshot JSON completo dos dados do pedido no instante da transição.
- **[RF-07] Despacho Assíncrono com Polling de 2 Segundos:** Um worker dedicado em processo independente deve consultar a tabela outbox a cada 2 segundos, selecionando lotes de eventos pendentes e disparando as requisições HTTP para os destinos cadastrados.
- **[RF-08] Consulta ao Histórico de Entregas:** A API deve permitir ao cliente consultar as últimas 100 tentativas de envio via `GET /webhooks/:id/deliveries`, exibindo payload, headers, código HTTP de resposta, tempo de resposta e status de entrega.
- **[RF-09] Rotação Segura de Credencial:** A API deve permitir a solicitação de rotação de secret via `POST /webhooks/:id/rotate-secret`, ativando uma nova credencial e mantendo a chave legada válida durante período de carência de 24 horas.
- **[RF-10] Replay Administrativo de DLQ:** A API deve fornecer endpoint `POST /admin/webhooks/dead-letter/:id/replay`, restrito exclusivamente ao perfil `ADMIN`, permitindo reenfileirar um evento morto na outbox para novo processamento, registrando usuário e timestamp de auditoria.
- **[RF-11] Assinatura Digital e Cabeçalhos Padronizados:** O worker deve enviar as requisições HTTP injetando obrigatoriamente:
  - `X-Event-Id`: identificador universal único (UUID) para desduplicação;
  - `X-Signature`: assinatura HMAC-SHA256 gerada sobre o corpo da mensagem;
  - `X-Timestamp`: epoch Unix do instante de envio para validação contra replay attack;
  - `X-Webhook-Id`: identificador cadastral do endpoint;
  - `Content-Type: application/json`.

---

## 7. Requisitos Não Funcionais

- **[RNF-01] Latência de Notificação:** O tempo total decorrido entre o commit da transação de status e o recebimento da requisição pelo cliente deve ser menor que 10 segundos para 99% dos disparos ($P99 < 10\text{s}$).
- **[RNF-02] Confiabilidade de Entrega (At-Least-Once):** Nenhum evento confirmado no banco de dados deve ser descartado sem que tenha havido 5 tentativas de entrega ou persistência definitiva na tabela de DLQ.
- **[RNF-03] Isolamento Transacional e Resiliência:** Lentidão, indisponibilidade ou falhas nos servidores dos clientes não podem, sob nenhuma hipótese, degradar a performance ou provocar rollback na API de pedidos do OMS.
- **[RNF-04] Transporte e Criptografia Mandatórios:** O sistema deve rejeitar o cadastro de URLs não criptografadas (`http://`), exigindo estritamente conexões seguras sob TLS (`https://`).
- **[RNF-05] Limite Máximo de Carga Útil:** O tamanho do payload JSON do webhook não deve ultrapassar 64KB. Requisições que excedam esse patamar devem ser rejeitadas com erro de validação.
- **[RNF-06] Timeout Rígido de Chamada Externa:** Toda chamada HTTP realizada pelo worker para os servidores dos clientes deve possuir timeout estrito de 10 segundos.
- **[RNF-07] Observabilidade Estruturada:** Todas as etapas de execução do worker e da API devem gerar logs estruturados em formato JSON utilizando o logger Pino existente, sem exposição de credenciais ou chaves secretas.

---

## 8. Decisões e Trade-offs Principais

### 8.1 Transacional Outbox no MySQL vs Disparo Síncrono
- **Decisão:** Rejeitar categoricamente o disparo HTTP síncrono no fluxo de pedidos e adotar o padrão Transacional Outbox no banco de dados existente.
- **Trade-off:** Assume-se a complexidade de manter uma tabela de eventos e um worker de polling em troca de garantia absoluta de consistência ACID, eliminação do problema da escrita dupla e blindagem da API de pedidos contra lentidões de terceiros.

### 8.2 Worker em Polling de 2s vs Plataforma Dedicada de Mensageria (Kafka / Redis)
- **Decisão:** Utilizar worker em Node.js consultando a tabela `webhook_outbox` a cada 2 segundos, descartando a introdução imediata de Kafka ou Redis Cluster.
- **Trade-off:** Aceita-se uma latência intrínseca de 0 a 2 segundos (perfeitamente tolerável frente ao requisito de 10s) e uma carga leve de leitura periódica no MySQL, em troca de zero custo adicional de infraestrutura, simplicidade operacional para o time e viabilidade de entrega no prazo de 3 sprints.

### 8.3 Semântica At-Least-Once com X-Event-Id vs Exactly-Once
- **Decisão:** Adotar garantia *at-least-once* combinada com o cabeçalho de identificação único `X-Event-Id`.
- **Trade-off:** Transfere-se para o integrador externo a responsabilidade de implementar desduplicação idempotente, eliminando a necessidade de protocolos distribuídos de coordenação complexos e lentos (como 2PC).

---

## 9. Dependências

- **Banco de Dados MySQL 8.x:** Suporte a transações ACID e tipos JSON nativos já provisionado em ambiente de produção.
- **Prisma ORM 5.x:** Utilizado para migração declarativa das novas tabelas (`webhook_outbox`, `webhook_dead_letter`) e manipulação de transações tipadas.
- **Node.js 20.x LTS & Express 4.x:** Ambiente de runtime da aplicação e framework HTTP já homologados.
- **Biblioteca Zod:** Utilizada para validação dos contratos de entrada da API.
- **Logger Pino:** Infraestrutura de logs estruturados em JSON já padronizada no repositório.
- **Equipe de Segurança da Informação:** Janela mandatória de 2 dias úteis reservada para auditoria do código de criptografia HMAC e geração de secrets antes do deploy em produção.

---

## 10. Riscos e Mitigação

### [Risco 1] Sobrecarga e Concorrência de Leitura no Banco de Dados MySQL
- **Probabilidade:** Média
- **Impacto:** Alto
- **Mitigação:** 
  1. Criação de índice composto otimizado cobrindo exatamente a consulta do worker: `INDEX idx_outbox_polling (status, next_retry_at, created_at)`.
  2. Limitação rigorosa de leitura em pequenos lotes (`LIMIT 50`) para evitar bloqueios de tabela.
  3. Implantação de rotina periódica de expurgo que remove registros entregues (`DELIVERED`) após 30 dias.

### [Risco 2] Vazamento de Chaves Secretas por Falha de Segurança no Cliente
- **Probabilidade:** Média
- **Impacto:** Alto
- **Mitigacao:**
  1. Geração de secrets com alta entropia criptográfica (32 bytes aleatórios).
  2. Isolamento de credenciais com chave estritamente individual por endpoint cadastrado.
  3. Disponibilização de endpoint de rotação de secret com período de carência de 24 horas, permitindo transição suave sem indisponibilidade.
  4. Mascaramento obrigatório das chaves em todos os endpoints de consulta.

### [Risco 3] Lentidão Crônica de Servidores de Clientes Esgotando Recursos do Worker
- **Probabilidade:** Alta
- **Impacto:** Médio
- **Mitigação:**
  1. Aplicação de timeout estrito e não negociável de 10 segundos em todas as chamadas externas.
  2. Isolamento do worker em processo separado do sistema operacional (`src/worker.ts`), impedindo que lentidões degradem o event loop da API principal.
  3. Descarte para fila de mensagens mortas (DLQ) após 5 tentativas falhas.

---

## 11. Critérios de Aceitação

- [CA-01] O cliente consegue cadastrar um endpoint de webhook via `POST /webhooks` com URL HTTPS válida e lista de status desejados, recebendo a secret gerada.
- [CA-02] Tentativas de cadastro com URLs inseguras (`http://`) ou dados malformados são rejeitadas com HTTP 400.
- [CA-03] A alteração de status de pedido em `changeStatus` persiste o snapshot do evento na tabela `webhook_outbox` atomicamente; em caso de rollback, o evento não é gerado.
- [CA-04] O worker consome a outbox em ciclos de 2 segundos e entrega o evento no destino em menos de 10 segundos ($P99 < 10\text{s}$).
- [CA-05] Toda requisição enviada ao cliente possui os cabeçalhos `X-Event-Id`, `X-Signature`, `X-Timestamp` e `X-Webhook-Id`.
- [CA-06] A assinatura enviada em `X-Signature` é validada com sucesso pelo cliente utilizando o algoritmo HMAC-SHA256 sobre o corpo da mensagem.
- [CA-07] Endpoints que apresentarem instabilidade sofrem até 5 tentativas com backoff progressivo e jitter (1m, 5m, 30m, 2h, 12h).
- [CA-08] Eventos que falharem após a 5ª tentativa são transferidos para a tabela `webhook_dead_letter`.
- [CA-09] O endpoint `POST /admin/webhooks/dead-letter/:id/replay` reprocessa o evento e exige obrigatoriamente a role `ADMIN`.
- [CA-10] O endpoint `GET /webhooks/:id/deliveries` retorna o histórico recente das tentativas de envio.

---

## 12. Estratégia de Testes e Validação

- **Testes Unitários:**
  - Validação dos schemas Zod para criação e edição de webhooks (rejeição de HTTP, URLs inválidas, eventos não existentes).
  - Testes do gerador criptográfico de secrets e do algoritmo de cálculo de assinatura HMAC-SHA256.
  - Teste da função de cálculo de backoff exponencial com aplicação de jitter.
- **Testes de Integração:**
  - Teste de atomicidade relacional: simular erro durante `changeStatus` e verificar que a inserção na `webhook_outbox` sofre rollback.
  - Teste do worker de polling: simular worker processando lotes de eventos pendentes e atualizando status para `DELIVERED`.
  - Teste da rota de replay da DLQ: validar rejeição de acesso com papel `OPERATOR` e sucesso com papel `ADMIN`.
- **Testes Ponta a Ponta (E2E) com Servidor Mock:**
  - Subir servidor mock local simulando o endpoint do cliente e validar o recebimento da notificação, conferência dos headers e validação da assinatura HMAC.
  - Simular timeout e respostas de erro 500 para validar as 5 retentativas e a posterior movimentação para a tabela `webhook_dead_letter`.
- **Auditoria de Segurança:**
  - Revisão de código de segurança conduzida pela engenheira Sofia, focando na proteção de secrets em repouso e integridade das assinaturas.
