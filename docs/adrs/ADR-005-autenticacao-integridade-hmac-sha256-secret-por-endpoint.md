# ADR-005: Autenticação e Integridade via HMAC-SHA256 com Secret por Endpoint
**Status:** Aceito
**Data:** 2026-09-13

**Related to:**
- [ADR-001: Padrão Transacional Outbox no MySQL](./ADR-001-padrao-transacional-outbox-no-mysql.md)
- [ADR-003: Garantia de Entrega At-Least-Once com Desduplicação por Event ID](./ADR-003-garantia-entrega-at-least-once-com-desduplicacao-event-id.md)
- [ADR-006: Reaproveitamento Integral dos Padrões da Codebase](./ADR-006-reaproveitamento-padroes-codebase.md)

## Contexto e Definição do Problema

O tráfego de notificações de pedidos pela internet pública envolve o transporte de informações operacionais sensíveis de clientes corporativos para servidores externos. Sem mecanismos criptográficos de segurança na camada de aplicação, os destinatários tornam-se vulneráveis a ataques de personificação por terceiros maliciosos e a ataques de repetição ou adulteração de conteúdo em trânsito.

A utilização de um segredo único global compartilhado para toda a plataforma representa um risco catastrófico, pois o comprometimento de uma única integração invalidaria a segurança de todos os demais clientes. Por outro lado, modelos baseados em infraestrutura de certificados digitais assimétricos impõem atrito operacional excessivo de emissão e renovação para clientes de médio porte. Além disso, a troca periódica de credenciais de integração exige mecanismos de convivência que evitem quedas no fluxo contínuo de notificações.

Faz-se mandatório definir a técnica criptográfica de autenticação e garantia de integridade das mensagens, a política de segregação de chaves de segurança e o procedimento de rotação suave de credenciais com período de carência.

## Direcionadores de Decisão

* Assegurar autenticidade da origem e integridade de conteúdo dos eventos contra adulterações em trânsito.
* Isolar o impacto de eventuais vazamentos restringindo cada segredo exclusivamente ao seu respectivo ponto de recebimento.
* Permitir a rotação programada de credenciais pelos clientes sem interrupção no recebimento de notificações.
* Adotar algoritmos padronizados suportados nativamente pelas principais linguagens e ferramentas do mercado.
* Prevenir ataques de repetição e esgotamento de recursos com cabeçalhos temporais e limites estritos de conteúdo.

## Opções Consideradas

* Assinatura criptográfica por código de autenticação de mensagem com segredo individual por destino e carência de rotação
* Chave secreta global única compartilhada com todos os clientes da plataforma
* Autenticação mútua na camada de transporte via certificados digitais cliente e servidor

## Resultado da Decisão

Opção escolhida: Assinatura criptográfica por código de autenticação de mensagem com segredo individual por destino e carência de rotação, porque oferece robustez criptográfica máxima na camada de aplicação com ampla interoperabilidade para os clientes corporativos, sem impor a sobrecarga operacional de certificados digitais.

A solução gera um segredo criptográfico independente de alta entropia para cada ponto de recebimento cadastrado. A assinatura é gerada utilizando o algoritmo seguro de dispersão sobre a cadeia literal serializada do corpo da mensagem e transmitida em cabeçalho padronizado da requisição, acompanhada por cabeçalho com carimbo temporal para combate a ataques de repetição e cabeçalho identificador do cadastro. A rotação de credenciais suporta uma janela de carência de vinte e quatro horas, período no qual a chave anterior permanece válida simultaneamente à nova chave, viabilizando transições seguras e sem paradas. Adicionalmente, impõe-se a obrigatoriedade de transporte criptografado e o teto máximo de sessenta e quatro quilobytes por mensagem.

## Prós e Contras das Opções

### Assinatura criptográfica por código de autenticação de mensagem com segredo individual por destino e carência de rotação

* Bom, porque atesta matematicamente a autenticidade da origem e a integridade de cada mensagem transmitida.
* Bom, porque isola riscos de segurança ao impedir que o vazamento de um segredo comprometa outros clientes ou integrações.
* Bom, porque a janela de transição de vinte e quatro horas permite aos clientes atualizar suas configurações sem perda de mensagens.
* Ruim, porque exige que a camada de persistência mantenha simultaneamente a credencial corrente e a anterior durante a janela de transição.

### Chave secreta global única compartilhada com todos os clientes da plataforma

* Bom, porque simplifica a geração, validação e armazenamento de credenciais no banco de dados.
* Ruim, porque qualquer vazamento acidental em um cliente compromete a integridade do sistema como um todo.
* Ruim, porque a rotação emergencial obriga a reconfiguração simultânea de todos os parceiros para não interromper os serviços.

### Autenticação mútua na camada de transporte via certificados digitais cliente e servidor

* Bom, porque fornece garantia estrita de identidade em nível de rede antes do envio do conteúdo da requisição.
* Ruim, porque adiciona forte fricção no processo de integração dos parceiros devido à gestão de chaves públicas e autoridades certificadoras.
* Ruim, porque eleva a complexidade de manutenção e renovação periódica de certificados para o time de suporte e infraestrutura.

## Consequências

A arquitetura assegura alto padrão de segurança para as notificações externas, protegendo os clientes corporativos contra ataques de personificação e modificações maliciosas em trânsito. A obrigatoriedade de conexões seguras é fiscalizada desde a validação inicial do cadastro, recusando protocolos abertos e barrando cargas úteis que excedam os limites previstos.

Para garantir a correspondência exata de bytes entre o conteúdo assinado e o que trafega na rede, o retrato textual estruturado da mensagem é consolidado e serializado na criação do evento, permitindo que a rotina de envio calcule a assinatura diretamente sobre a cadeia imutável. A governança de segurança prevê ainda revisão técnica especializada antes da implantação das rotinas de geração e validação de chaves.

## Referências

* `src/config/env.ts:1`
