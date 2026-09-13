# ADR-006: Reaproveitamento Integral dos Padrões da Codebase
**Status:** Aceito
**Data:** 2026-09-13
**ADRs Relacionados:** ADR-001, ADR-002, ADR-004, ADR-005

## Contexto e Definição do Problema

A implementação de uma nova capacidade assíncrona de notificações demanda a criação de modelos de dados, rotinas de consumo, endpoints de configuração para clientes corporativos e interfaces operacionais de administração. Em equipes enxutas de engenharia, a introdução de novos paradigmas arquiteturais, bibliotecas redundantes ou arquiteturas de serviços desacoplados em novos repositórios costuma introduzir fragmentação tecnológica e despesas operacionais não planejadas.

A base de código da plataforma já conta com padrões arquiteturais consolidados em seus domínios essenciais, incluindo organização modular homogênea, tipagem de contratos e validações declarativas na entrada, mapeamento de banco de dados relacional via mapeador existente, tipagem unificada de erros operacionais e formatação padronizada de registros de auditoria e monitoramento.

Dessa forma, faz-se necessário definir a diretriz de engenharia para o desenvolvimento do novo módulo de notificações, estabelecendo se a capacidade deve seguir estritamente as convenções vigentes da base de código ou adotar modelos e tecnologias independentes.

## Direcionadores de Decisão

* Manter a homogeneidade arquitetural e a uniformidade de convenções em toda a base de código do sistema.
* Maximizar o reaproveitamento de componentes transversais existentes de validação, erro, segurança e registros.
* Reduzir o tempo de entrega e a curva de aprendizado da equipe técnica para sustentar e evoluir o novo módulo.
* Evitar custos operacionais e redundâncias decorrentes de novas ferramentas ou infraestruturas segregadas de compilação e publicação.
* Garantir manutenibilidade e facilidade de rastreamento com prefixos padronizados de códigos de exceção.

## Opções Consideradas

* Adoção estrita dos padrões arquiteturais modulares e transversais existentes da base de código
* Construção do sistema de notificações como microserviço isolado com pilha tecnológica independente
* Implementação direta nos domínios existentes utilizando bibliotecas e convenções de tratamento avulsas

## Resultado da Decisão

Opção escolhida: Adoção estrita dos padrões arquiteturais modulares e transversais existentes da base de código, porque acelera o ciclo de desenvolvimento garantindo a entrega da funcionalidade dentro do prazo estimado de três ciclos, preservando a coerência estrutural e facilitando a manutenção futura por qualquer engenheiro da equipe.

O desenvolvimento da funcionalidade é estruturado como um novo módulo coeso no diretório padrão de módulos, mantendo a segregação estrita em camadas de controle de requisições, regras de negócio, acesso a dados, definição de rotas e esquemas de validação declarativa. O tratamento de exceções utiliza exclusivamente a classe base padronizada da plataforma, adotando códigos de erro de máquina prefixados para identificar inequivocamente o domínio nas respostas e registros. São reaproveitados integralmente o interceptador centralizado de erros, os componentes de controle de acesso para proteção de ações sensíveis e a infraestrutura unificada de registros estruturados.

## Prós e Contras das Opções

### Adoção estrita dos padrões arquiteturais modulares e transversais existentes da base de código

* Bom, porque zera a curva de adaptação da equipe ao manter estrutura idêntica à dos demais módulos do sistema.
* Bom, porque reaproveita rotinas consolidadas de validação, mapeamento de banco, tratamento de erros e autenticação.
* Bom, porque possibilita a conclusão segura do desenvolvimento dentro do cronograma planejado de três ciclos.
* Ruim, porque exige estrita disciplina para respeitar convenções estabelecidas e evitar atalhos informais de codificação.

### Construção do sistema de notificações como microserviço isolado com pilha tecnológica independente

* Bom, porque permite escolher pilhas especializadas para processamento intensivo de eventos concorrentes.
* Ruim, porque multiplica a complexidade operacional com novas esteiras de entrega contínua, imagens e configurações de rede.
* Ruim, porque dilui o foco da equipe de engenharia com governança de repositórios múltiplos para uma funcionalidade pontual.

### Implementação direta nos domínios existentes utilizando bibliotecas e convenções de tratamento avulsas

* Bom, porque concede flexibilidade para experimentação de pacotes alternativos de registro e tratamento pontual.
* Ruim, porque fragmenta a forma como o sistema intercepta exceções e expõe inconsistências nas respostas aos clientes.
* Ruim, porque dificulta auditorias de segurança e eleva a complexidade de depuração em ambientes de produção.

## Consequências

A decisão reforça a padronização e a sustentabilidade de longo prazo do repositório, garantindo que o ciclo de vida do novo módulo de notificações seja compreendido e mantido por qualquer desenvolvedor da equipe sem necessidade de treinamento em ferramentas novas. A integração das novas rotas ocorre de maneira harmoniosa aos pontos de agregação de serviços da aplicação.

Adicionalmente, a consistência de códigos prefixados de erro simplifica a integração dos clientes corporativos e a análise de incidentes pelas equipes de suporte. A governança do projeto consolida o princípio de parcimônia tecnológica, estabelecendo que novos módulos devem sempre estender e valorizar as fundações arquiteturais já validadas na plataforma.

## Referências

* `src/shared/errors/app-error.ts:3`
* `src/middlewares/error.middleware.ts:14`
* `src/middlewares/auth.middleware.ts:49`
* `src/shared/logger/index.ts:1`
