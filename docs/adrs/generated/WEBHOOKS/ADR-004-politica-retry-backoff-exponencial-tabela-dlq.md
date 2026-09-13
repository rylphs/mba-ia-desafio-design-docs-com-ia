# ADR-004: Política de Retry com Backoff Exponencial e Tabela DLQ Dedicada
**Status:** Aceito
**Data:** 2026-09-13
**ADRs Relacionados:** ADR-001, ADR-002, ADR-006

## Contexto e Definição do Problema

O envio de notificações assíncronas a sistemas externos está constantemente exposto a falhas operacionais, tais como manutenções planejadas de clientes, saturação de servidores receptores ou quedas temporárias de conectividade. O tratamento dessas falhas exige um mecanismo estruturado que evite tanto o descarte precipitado de mensagens legítimas quanto a sobrecarga contínua da infraestrutura dos parceiros.

Políticas de retentativa agressivas com intervalos curtos esgotam tentativas rapidamente durante manutenções comuns de duas horas, ocasionando perdas indevidas. Em contrapartida, retentativas indefinidas provocam acúmulo perpétuo de registros não entregues no repositório de eventos ativos, degradando o rendimento do consumo periódico. Adicionalmente, eventos que esgotarem todas as tentativas automáticas necessitam de segregação física para auditoria, depuração e reprocessamento sob demanda.

Faz-se mandatório definir a disciplina de espaçamento das tentativas de entrega, o teto máximo de retransmissões automáticas e a estratégia de armazenamento isolado para eventos com falha definitiva, incluindo o mecanismo seguro de reprocessamento operacional.

## Direcionadores de Decisão

* Suportar janelas usuais de indisponibilidade ou manutenção prolongada de parceiros externos sem perda de eventos.
* Prevenir sobrecarga na infraestrutura dos clientes decorrente de retransmissões imediatas ou muito frequentes.
* Isolar mensagens com falha permanente para não onerar o repositório principal de eventos pendentes.
* Fornecer capacidade de auditoria detalhada e reprocessamento manual controlado de falhas definitivas.
* Restringir o acesso a operações de reprocessamento a perfis administrativos com registro de auditoria.

## Opções Consideradas

* Retentativas com espaçamento exponencial limitado e segregação em repositório dedicado de mensagens mortas
* Retentativas frequentes em intervalos curtos e fixos sem segregação física de falhas
* Retentativas contínuas por tempo indeterminado mantidas no repositório principal de eventos

## Resultado da Decisão

Opção escolhida: Retentativas com espaçamento exponencial limitado e segregação em repositório dedicado de mensagens mortas, porque equilibra tolerância a falhas temporárias com a governança e proteção dos sistemas integrados. A estratégia adota o limite estrito de cinco tentativas automáticas espaçadas progressivamente em intervalos de um minuto, cinco minutos, trinta minutos, duas horas e doze horas.

Essa progressão cobre uma janela operacional de aproximadamente quinze horas entre o primeiro incidente e a última tentativa, permitindo que manutenções e indisponibilidades convencionais sejam superadas sem intervenção manual. Em caso de esgotamento das tentativas ou tempo esgotado na comunicação, o evento é transferido para um repositório dedicado de mensagens mortas contendo o conteúdo estruturado original, a justificativa da falha e o registro cronológico. O reprocessamento é viabilizado por meio de rota administrativa restrita exclusivamente a perfis de administração, com gravação de auditoria do executor e retorno do evento ao fluxo inicial como pendente.

## Prós e Contras das Opções

### Retentativas com espaçamento exponencial limitado e segregação em repositório dedicado de mensagens mortas

* Bom, porque cobre janelas de indisponibilidade de até quinze horas, acomodando manutenções programadas de clientes.
* Bom, porque previne tempestades de requisições sobre servidores em recuperação ao expandir progressivamente os intervalos.
* Bom, porque mantém o repositório principal de despacho limpo e otimizado para consultas periódicas de alto desempenho.
* Ruim, porque requer gestão de modelos de dados adicionais e rotinas administrativas específicas para reprocessamento manual.

### Retentativas frequentes em intervalos curtos e fixos sem segregação física de falhas

* Bom, porque agiliza a resolução de falhas de rede de curtíssima duração em poucos segundos.
* Ruim, porque esgota prematuramente todas as tentativas caso o cliente enfrente uma manutenção superior a trinta minutos.
* Ruim, porque intensifica a carga em sistemas externos que já se encontram instáveis ou sobrecarregados.

### Retentativas contínuas por tempo indeterminado mantidas no repositório principal de eventos

* Bom, porque teoricamente garante a entrega eventual sem necessidade de descarte ou intervenção manual.
* Ruim, porque polui indefinidamente a base ativa de eventos com notificações destinadas a endpoints desativados ou extintos.
* Ruim, porque degrada progressivamente a eficiência das consultas de despacho do processador assíncrono.

## Consequências

A implementação estabelece um padrão robusto de resiliência e observabilidade para a entrega de notificações assíncronas. Os operadores e engenheiros ganham visibilidade detalhada sobre parceiros que apresentam falhas persistentes, podendo analisar o motivo específico do erro e reprocessar eventos individualmente após o restabelecimento da conectividade pelo cliente.

O repositório dedicado de falhas permanece preservado para consulta e rastreamento histórico, operando de forma desvinculada do ciclo de vida de expurgo de trinta dias aplicado aos eventos entregues com êxito na base principal. A proteção da rota de reprocessamento por perfil administrativo assegura conformidade de auditoria e impede reenvios acidentais por agentes não autorizados.

## Referências

* `src/middlewares/auth.middleware.ts:49`
* `docs/adrs/must-include.md:8`
