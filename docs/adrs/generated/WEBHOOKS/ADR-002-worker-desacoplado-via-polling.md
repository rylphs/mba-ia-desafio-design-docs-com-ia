# ADR-002: Worker Desacoplado via Polling
**Status:** Aceito
**Data:** 2026-09-13

**Depends on:** [ADR-001: Padrão Transacional Outbox no MySQL](./ADR-001-padrao-transacional-outbox-no-mysql.md)

**Related to:**
- [ADR-004: Política de Retry com Backoff Exponencial e Tabela DLQ Dedicada](./ADR-004-politica-retry-backoff-exponencial-tabela-dlq.md)
- [ADR-006: Reaproveitamento Integral dos Padrões da Codebase](./ADR-006-reaproveitamento-padroes-codebase.md)

## Contexto e Definição do Problema

Com a adoção do padrão transacional outbox para a persistência atômica de eventos de notificação no banco de dados relacional, faz-se necessário definir a estratégia de leitura, processamento e despacho dessas mensagens para os servidores dos clientes externos.

Executar o consumo assíncrono dentro do mesmo processo em que opera a API principal apresenta riscos de esgotamento do ciclo de eventos por operações de rede lentas, além de causar interrupção do envio de notificações sempre que a API for reiniciada para atualizações ou manutenções. Por outro lado, o mecanismo de busca dos eventos pendentes deve garantir baixa latência e preservar a sequência de atualizações de cada pedido sem introduzir tecnologias complexas de mensageria.

Torna-se mandatório definir a arquitetura de execução do processador de notificações e o mecanismo de consumo dos eventos gravados, assegurando que o intervalo de envio satisfaça o requisito comercial de latência inferior a dez segundos acordado com parceiros estratégicos.

## Direcionadores de Decisão

* Isolar os recursos de processamento assíncrono do tráfego de requisições da interface HTTP principal.
* Cumprir o limite contratual de latência de entrega abaixo de dez segundos para notificações em tempo real.
* Manter a ordem cronológica de eventos para o mesmo pedido em ambiente de processamento único.
* Assegurar resiliência do processamento assíncrono contra falhas ou reinicializações do servidor web.
* Evitar custos de infraestrutura e sobrecarga operacional com tecnologias externas de mensageria ou gatilhos complexos.

## Opções Consideradas

* Processador assíncrono desacoplado em processo dedicado com consulta periódica por intervalos
* Processador assíncrono executado dentro do mesmo ciclo de vida e processo do servidor web principal
* Consumo reativo disparado por gatilhos do banco de dados com acionamento externo

## Resultado da Decisão

Opção escolhida: Processador assíncrono desacoplado em processo dedicado com consulta periódica por intervalos, porque garante total independência operacional entre o envio de notificações e as requisições recebidas pela interface pública. A inicialização ocorre por meio de ponto de entrada exclusivo no sistema, mantendo seu próprio gerenciamento de conexões com o banco de dados e controle de encerramento gracioso para não interromper despachos em voo.

O processamento adota consulta periódica em intervalos regulares de dois segundos para leitura de pequenos lotes de eventos pendentes ordenados por data de criação. Essa abordagem atende com folga à meta de latência inferior a dez segundos exigida pelos clientes. Inicialmente, o sistema opera com uma única instância consumidora, o que assegura ordenação sequencial por pedido sem a necessidade imediata de algoritmos distribuídos de particionamento.

## Prós e Contras das Opções

### Processador assíncrono desacoplado em processo dedicado com consulta periódica por intervalos

* Bom, porque isola o consumo de recursos computacionais e previne que picos de chamadas externas afetem a disponibilidade da API.
* Bom, porque assegura que reinicializações do servidor web não interrompam as rotinas assíncronas de envio.
* Bom, porque entrega latência previsível inferior a dez segundos com baixo consumo de recursos através de lotes pequenos.
* Ruim, porque impõe consultas regulares contínuas ao banco de dados relacional mesmo durante períodos de inatividade.

### Processador assíncrono executado dentro do mesmo ciclo de vida e processo do servidor web principal

* Bom, porque simplifica a esteira de implantação utilizando uma única rotina de inicialização de processo.
* Ruim, porque sobrecarrega o mesmo laço de eventos com requisições HTTP externas e chamadas de clientes remotos.
* Ruim, porque interrupções, reinicializações ou quedas do servidor web paralisam imediatamente o envio de notificações.

### Consumo reativo disparado por gatilhos do banco de dados com acionamento externo

* Bom, porque elimina a necessidade de consultas periódicas em momentos sem novos eventos cadastrados.
* Ruim, porque a tecnologia de banco de dados utilizada carece de mecanismos nativos de escuta e notificação reativa de processos.
* Ruim, porque acoplamentos artificiais entre gatilhos de banco e serviços externos comprometem a confiabilidade e a segurança do banco.

## Consequências

A decisão desacopla completamente a esteira de envio de notificações do tráfego de usuários da plataforma, impedindo que falhas em servidores de terceiros afetem a capacidade de resposta das transações de pedidos. A governança de implantação passa a contar com dois processos distintos em produção, cada qual com monitoramento de integridade e ciclo de encerramento controlado via sinais do sistema operacional.

A garantia de ordenação dos eventos por pedido permanece condicionada ao funcionamento de uma instância única consumidora. Caso a volumetria futura exija paralelização horizontal, a arquitetura deverá evoluir para técnicas de particionamento de pedidos ou controle concorrente por bloqueio de registros em lote.

## Referências

* `src/server.ts:6`
* `src/config/database.ts:7`
* `package.json:14`
