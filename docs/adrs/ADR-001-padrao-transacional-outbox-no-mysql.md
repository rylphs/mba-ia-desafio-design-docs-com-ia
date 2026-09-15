# ADR-001: Padrão Transacional Outbox no MySQL
**Status:** Aceito
**Data:** 2026-09-13

**Used by:**
- [ADR-002: Worker Desacoplado via Polling](./ADR-002-worker-desacoplado-via-polling.md)
- [ADR-003: Garantia de Entrega At-Least-Once com Desduplicação por Event ID](./ADR-003-garantia-entrega-at-least-once-com-desduplicacao-event-id.md)
- [ADR-004: Política de Retry com Backoff Exponencial e Tabela DLQ Dedicada](./ADR-004-politica-retry-backoff-exponencial-tabela-dlq.md)

**Related to:**
- [ADR-005: Autenticação e Integridade via HMAC-SHA256 com Secret por Endpoint](./ADR-005-autenticacao-integridade-hmac-sha256-secret-por-endpoint.md)
- [ADR-006: Reaproveitamento Integral dos Padrões da Codebase](./ADR-006-reaproveitamento-padroes-codebase.md)

## Contexto e Definição do Problema

A plataforma atende clientes corporativos que necessitam acompanhar atualizações de pedidos em tempo real. Três grandes clientes parceiros reportaram ineficiências operacionais severas decorrentes de consultas periódicas por varredura à API de pedidos para verificar alterações de status. Há exigência comercial formal para disponibilização de notificações ativas de mudança de ciclo de vida com latência inferior a dez segundos até o término do trimestre, sob risco explícito de perda de contrato relevante para a concorrência.

Para suprir essa demanda sem comprometer a confiabilidade do sistema, é imprescindível estabelecer uma comunicação assíncrona entre o encadeamento de pedidos e os destinatários externos. Disparos diretos de requisições de rede durante a execução das regras de negócio expõem o processamento de pedidos a travamentos provocados pela lentidão de servidores remotos, além de inviabilizar a integridade transacional de cancelamento em cenários de indisponibilidade transitória do destinatário.

Portanto, faz-se necessário definir uma arquitetura capaz de assegurar consistência estrita entre o registro das modificações de domínio e a emissão dos respectivos eventos de notificação. A solução precisa eliminar o risco de inconsistência de escrita dupla sem introduzir sobrecarga de manutenção de novas tecnologias de mensageria para a equipe de engenharia.

## Direcionadores de Decisão

* Garantir consistência atômica entre a atualização dos pedidos e a criação de eventos, eliminando o problema de escrita dupla.
* Atender ao prazo contratual do trimestre com entrega de notificações em latência inferior a dez segundos.
* Preservar a estabilidade e o tempo de resposta do processamento de pedidos contra instabilidades e lentidões externas.
* Evitar custos operacionais e provisionamento de novos componentes de infraestrutura mantendo a base tecnológica existente.
* Assegurar a fidelidade histórica do evento persistindo o retrato completo dos dados no momento exato da alteração de estado.

## Opções Consideradas

* Padrão Transacional Outbox no banco de dados relacional existente
* Disparo síncrono de notificações via rede no fluxo de atualização do pedido
* Publicação direta de eventos em serviço dedicado de mensageria externa

## Resultado da Decisão

Opção escolhida: Padrão Transacional Outbox no banco de dados relacional existente, porque viabiliza a persistência atômica do evento de notificação no mesmo escopo transacional das alterações de dados do pedido e movimentações de inventário. Caso a transação seja confirmada, a existência durável do evento está assegurada; caso ocorra reversão por inconsistência de domínio, o evento é descartado conjuntamente.

A persistência adota identificadores universais e registra o retrato completo do evento em formato textual estruturado no momento da gravação, blindando o conteúdo contra mutações futuras do pedido. Essa abordagem atende à meta de entrega rápida e viabiliza a entrega dentro do cronograma previsto de três ciclos de desenvolvimento, incorporando retenção de eventos entregues por trinta dias antes de arquivamento.

## Prós e Contras das Opções

### Padrão Transacional Outbox no banco de dados relacional existente

* Bom, porque garante consistência transacional absoluta eliminando cenários de escrita dupla ou eventos órfãos.
* Bom, porque reaproveita a infraestrutura existente de persistência relacional sem custos adicionais de gerenciamento.
* Bom, porque desacopla a latência e a disponibilidade das integrações externas do fluxo principal de pedidos.
* Ruim, porque impõe carga adicional de operações no banco de dados principal e demanda rotinas para arquivamento periódico.

### Disparo síncrono de notificações via rede no fluxo de atualização do pedido

* Bom, porque possui concepção inicial direta sem dependência de tabelas auxiliares de despacho.
* Ruim, porque lentidões em sistemas externos degradam a disponibilidade e o tempo de resposta das transações de pedidos.
* Ruim, porque falhas de rede após a gravação local impedem reversões limpas e geram inconsistência entre os sistemas.
* Ruim, porque carece de mecanismos nativos de resiliência e retentativas para entregas não sucedidas.

### Publicação direta de eventos em serviço dedicado de mensageria externa

* Bom, porque fornece recursos nativos de distribuição assíncrona com alto rendimento de transferência.
* Ruim, porque reintroduz o risco de escrita dupla caso a confirmação do banco ocorra com falha posterior na rede do serviço de mensageria.
* Ruim, porque demanda provisionamento de novos clusters e eleva a complexidade de manutenção operacional para o time.

## Consequências

A escolha assegura confiabilidade estrita na notificação de eventos e mantém o ciclo de pedidos desacoplado da estabilidade dos destinatários. A retenção do estado integral do evento no momento da emissão garante conformidade de auditoria e simplifica a integração com consumidores externos, garantindo o cumprimento dos prazos contratuais acordados com parceiros estratégicos sem elevação de custo operacional com novos servidores.

Em contrapartida, a persistência relacional passa a absorver a volumetria de escrita dos eventos e leituras concorrentes para consumo. Essa sobrecarga é mitigada por meio de indexação eficiente por situação de entrega e ordem cronológica, limitando o consumo a pequenos lotes pendentes e aplicando expurgo programado de registros entregues após trinta dias.

## Referências

* `src/modules/orders/order.service.ts:131`
* `prisma/schema.prisma:74`
* `src/config/database.ts:7`
