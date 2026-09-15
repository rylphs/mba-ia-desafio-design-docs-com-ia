# ADR-003: Garantia de Entrega At-Least-Once com Desduplicação por Event ID
**Status:** Aceito
**Data:** 2026-09-13

**Depends on:** [ADR-001: Padrão Transacional Outbox no MySQL](./ADR-001-padrao-transacional-outbox-no-mysql.md)

**Related to:**
- [ADR-004: Política de Retry com Backoff Exponencial e Tabela DLQ Dedicada](./ADR-004-politica-retry-backoff-exponencial-tabela-dlq.md)
- [ADR-005: Autenticação e Integridade via HMAC-SHA256 com Secret por Endpoint](./ADR-005-autenticacao-integridade-hmac-sha256-secret-por-endpoint.md)

## Contexto e Definição do Problema

O envio de notificações assíncronas via chamadas de rede pela internet pública está sujeito a instabilidades de conectividade, indisponibilidades temporárias de servidores destinatários e interrupções de conexão por tempo esgotado. Nesses cenários, é imperativo estabelecer o contrato de garantia de entrega de dados entre a plataforma e os sistemas integradores externos.

Tentativas de garantir entrega estritamente única requerem coordenação e confirmação mútua em duas fases entre sistemas heterogêneos, o que eleva drasticamente a complexidade da integração, aumenta a latência de comunicação e degrada a tolerância a partições de rede. Por outro lado, modelos sem retentativas podem acarretar perda definitiva de eventos cruciais de mudança de ciclo de vida de pedidos, quebrando a premissa de negócio exigida pelos clientes corporativos.

Dessa forma, faz-se necessário definir o modelo formal de entrega de mensagens adotado pelo sistema de notificações, além dos mecanismos padronizados de identificação que permitirão aos clientes diferenciar despachos legítimos de eventuais retransmissões.

## Direcionadores de Decisão

* Eliminar a possibilidade de perda de notificações em caso de instabilidades transitórias de rede.
* Adotar padrão consolidado da indústria para integrações assíncronas entre sistemas corporativos.
* Estabelecer identificadores únicos duráveis para possibilitar desduplicação eficiente no consumidor.
* Manter a arquitetura interna simples e com baixo acoplamento em relação aos sistemas clientes.
* Documentar claramente o contrato de integração e as responsabilidades da camada receptora.

## Opções Consideradas

* Garantia de entrega ao menos uma vez com cabeçalho identificador único para desduplicação no destinatário
* Garantia de entrega estritamente única com protocolo bilateral de confirmação transacional
* Disparo único de notificação sem retransmissão em caso de falha de comunicação

## Resultado da Decisão

Opção escolhida: Garantia de entrega ao menos uma vez com cabeçalho identificador único para desduplicação no destinatário, porque assegura que nenhuma notificação de pedido seja perdida diante de problemas de rede ou lentidão temporária do receptor, alinhando a solução aos padrões amplamente estabelecidos no mercado corporativo.

Para viabilizar a desduplicação na ponta receptora, cada evento recebe um identificador universal exclusivo gerado no momento da gravação atômica da outbox. Esse identificador é transmitido tanto no cabeçalho padronizado da requisição quanto no corpo estruturado da mensagem, acompanhado pelo carimbo de data e hora do envio e pela identificação do cadastro de notificação. Cabe ao cliente registrar e verificar esse identificador em sua camada receptora, retendo o histórico de controle por ao menos vinte e quatro horas para cobrir com segurança a janela total de retentativas da plataforma.

## Prós e Contras das Opções

### Garantia de entrega ao menos uma vez com cabeçalho identificador único para desduplicação no destinatário

* Bom, porque assegura resiliência de entrega sem risco de descarte ou perda de notificações críticas de pedidos.
* Bom, porque adota prática padrão amplamente aceita por sistemas de integração e clientes corporativos.
* Bom, porque preserva a simplicidade e a escalabilidade da plataforma ao delegar o controle de estado idempotente ao receptor.
* Ruim, porque exige que os clientes externos implementem lógica de persistência e verificação de identificadores para evitar duplicidades.

### Garantia de entrega estritamente única com protocolo bilateral de confirmação transacional

* Bom, porque alivia os sistemas integradores da obrigação de controlar duplicidade de requisições.
* Ruim, porque exige coordenação transacional bilateral inviável em comunicações baseadas na internet pública aberta.
* Ruim, porque introduz alta latência e pontos únicos de travamento durante a confirmação mútua de recebimento.

### Disparo único de notificação sem retransmissão em caso de falha de comunicação

* Bom, porque simplifica a implementação eliminando políticas de retentativa e armazenamento de falhas.
* Ruim, porque qualquer oscilação de conectividade ou indisponibilidade momentânea do cliente causa perda definitiva do evento.
* Ruim, porque descumpre frontalmente os requisitos de confiabilidade comercial contratados com os parceiros.

## Consequências

A decisão formaliza o modelo operacional de notificações, delegando a responsabilidade de idempotência aos destinatários através do uso obrigatório do identificador universal fornecido em cada requisição. Essa exigência é documentada no portal de integração para desenvolvedores, orientando os parceiros sobre a retenção mínima recomendada dos identificadores recebidos por um período de vinte e quatro horas e sobre a necessidade de retorno de confirmação bem-sucedida para validar o recebimento.

Internamente, a plataforma garante a persistência imutável do identificador único desde a geração do evento no fluxo transacional de pedidos até o seu despacho final, simplificando os processos de diagnóstico, rastreamento de entregas e auditoria de integrações.

## Referências

* `package.json:32`
* `docs/adrs/must-include.md:7`
