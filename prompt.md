# Tarefa: Mapeamento de Origem e Persistência de Dados — IG3 e XF8

## Contexto
As aplicações das siglas IG3 e XF8 NÃO são Golden Source. Todos os dados são consultados em APIs Produto e/ou Datamesh, e depois persistidos em bases de dados próprias de cada sigla. A persistência pode ser:
(a) direta, no mesmo fluxo síncrono da consulta, ou
(b) indireta, quando o resultado da consulta vira um evento (SQS/SNS/Kafka) consumido por outro componente que faz a persistência.

Além disso, existem Glue Jobs que consultam Datamesh e persistem dados nas bases da sigla, fora do fluxo das APIs.

## Importante: o que NÃO é objetivo desta análise
**Não é objetivo mapear os endpoints/contratos que as aplicações IG3 e XF8 expõem** (ou seja, a API de entrada delas, o que elas oferecem para outros consumirem). O objeto do mapeamento é sempre **para fora e para dentro**: o que a aplicação **consulta** (em APIs Produto/Datamesh) e onde **persiste** o resultado.

## Conceito de "estímulo" (ponto de partida do fluxo)
Cada fluxo de dado começa a partir de um estímulo — o evento que aciona a aplicação a fazer a consulta. O estímulo pode ser, por exemplo:
- Uma chamada HTTP recebida (inbound)
- Consumo de uma mensagem SQS
- Consumo de um evento Kafka
- Um trigger agendado (schedule/cron)
- Outro evento assíncrono (SNS, etc.)

O estímulo deve ser identificado e citado **apenas como ponto de entrada do fluxo** (para saber onde no código a investigação começa), mas o detalhamento completo (parâmetros, headers, contrato) é exigido para a **consulta de saída** (API Produto/Datamesh) e para a **persistência**, não para o estímulo em si. Documentar o estímulo de forma resumida: tipo (HTTP/SQS/Kafka/schedule), nome/tópico/rota, e o que dispara o fluxo — sem se aprofundar no contrato de entrada.

## Branch de análise
Toda a investigação deve ser feita a partir da branch `develop` de cada repositório (fazer checkout/pull da `develop` antes de iniciar a análise — não usar `main`/`master` nem branches de feature). Registrar no relatório final o hash do commit analisado (`git rev-parse HEAD`) e confirmação da branch (`git branch --show-current`).

## Escopo por sigla

**IG3**
- Saldos
- Limites de contas

**XF8**
- Lançamentos Pendentes
- Lançamentos Efetivos
- Recebíveis

## Objetivo
Mapear, para cada sigla, a partir de cada estímulo relevante: o que é consultado (API Produto/Datamesh), quais transformações o dado sofre, e onde/como ele é persistido — com toda afirmação lastreada em evidência verificável no código, não em inferência.

## O que investigar

### 1. Estímulo (ponto de partida do fluxo)
Para cada fluxo relevante identificado, documentar de forma resumida:
- Tipo de estímulo (HTTP inbound, consumo SQS, consumo Kafka, schedule, SNS, etc.)
- Identificação (rota HTTP, nome do tópico/fila/schedule)
- Classe/método que recebe o estímulo (ponto de entrada no código)
- Não é necessário detalhar contrato completo do estímulo (isso não é objetivo da análise) — só o suficiente para rastrear qual fluxo de consulta/persistência ele dispara

### 2. Consultas de saída (chamadas a APIs Produto / Datamesh)
Para cada chamada identificada no código, a partir de um estímulo mapeado, documentar:
- Nome/responsabilidade da aplicação que faz a chamada
- Domínio de dado (conforme escopo da sigla)
- Endpoint completo (host + path) e verbo HTTP **da API consultada** (não da aplicação IG3/XF8)
- Parâmetros de request (path, query, body) com nome e tipo
- Headers enviados (incluindo autenticação, correlation-id, etc.)
- Estrutura do payload de resposta (campos, tipos, exemplos se houver)
- Client/biblioteca usado para a chamada (Feign, WebClient, RestTemplate, etc.) e onde está definido no código

### 3. Regras de transformação
Sempre que o dado sofrer qualquer transformação entre a consulta e a persistência (cálculo, agregação, enriquecimento, conversão, filtro, default de valor, etc.):
- Descrever a regra de negócio em linguagem clara (o quê e por quê, não só o código)
- Indicar o trecho/classe/método onde a regra está implementada
- Se a regra depender de configuração externa (feature flag, parâmetro de sistema), indicar isso
- Se não houver transformação (persistência é um espelho direto do retorno), declarar explicitamente "sem transformação — persistência 1:1"

### 4. Persistência (direta ou via evento)
Para cada destino de persistência, documentar obrigatoriamente:
- **Nome da base de dados**
- **Tipo da base** (PostgreSQL, DynamoDB, Oracle, etc.)
- **Nome da tabela/coleção**
- **Estrutura completa** (colunas, tipos, chaves primárias/estrangeiras, índices relevantes)
- Mapeamento campo de origem (resposta da API/evento) → coluna da tabela, já refletindo a regra de transformação aplicada (item 3)
- Camada/classe responsável pela persistência (repository, DAO, etc.)
- Se a persistência for via evento (SQS/SNS/Kafka): nome do tópico/fila, estrutura do payload publicado, e o consumidor responsável por persistir (sinalizar "não encontrado neste repo" se aplicável)

**Regra de tabela compartilhada:** se mais de um fluxo (ex: duas APIs diferentes, ou uma API e um Glue Job) persistir na mesma tabela, isso deve ficar explícito, incluindo:
- Todos os fluxos identificados que escrevem naquela tabela
- Quais colunas cada fluxo popula/atualiza
- Se há risco de conflito ou sobrescrita entre os fluxos, e como isso é tratado (ex: campo de origem, timestamp de atualização, upsert por chave)

### 5. Glue Jobs
Para cada Glue Job identificado (o estímulo aqui normalmente é o próprio schedule/trigger do job):
- Nome do job e trigger (schedule, evento, manual)
- Query/fonte no Datamesh (tabela, database, formato — Athena/Glue Catalog)
- Domínio de dado que ele processa
- Regras de transformação aplicadas (mesmo padrão do item 3)
- Nome da base, tipo da base, nome da tabela e estrutura completa do destino (mesmo padrão do item 4)
- Se escreve em tabela compartilhada com outro fluxo, aplicar a mesma regra do item 4

## Sensores e Validações
Toda afirmação factual do relatório deve passar por um sensor determinístico antes de ser incluída como confirmada. Não é permitido documentar algo com base em "parece ser" ou "provavelmente é" — se o sensor não confirmar, a afirmação vai para "Gaps/Observações" como não verificada.

| Tipo de afirmação | Sensor / comando de verificação | Critério de aprovação |
|---|---|---|
| Estímulo (rota/tópico de entrada) | `grep -r` pela rota/tópico/nome do listener no código | String encontrada, com caminho registrado |
| Endpoint consultado (API Produto/Datamesh) | `grep -r` pelo path literal no client/config | String encontrada no arquivo, com caminho registrado |
| Nome de tabela | Busca no DDL/migration/schema (ex: Flyway, Liquibase, `.sql`) | Definição da tabela encontrada, não apenas referência em query/repository |
| Estrutura da tabela (colunas/tipos) | Extração direta do DDL/migration, não da entity/DTO do código de aplicação | Estrutura extraída bate com a entity mapeada (se houver divergência, registrar) |
| Tópico/fila (SNS/SQS/Kafka) | Busca em arquivos de configuração/infra (application.yml, Terraform, CDK, CloudFormation) | Nome do recurso confirmado fora do código Java/aplicação |
| Glue Job | Busca em definição de infra (Terraform/CDK) ou export de configuração do job | Script/job encontrado com nome e fonte/destino explícitos |
| Campo em "Mapeamento campo → coluna" | Cruzamento: campo precisa existir tanto na resposta da API/evento (item 2) quanto na estrutura da tabela (item 4) | Ambos os lados confirmados; senão, sinalizar inconsistência |
| Tabela "compartilhada" | Encontrar evidência de escrita (insert/update/save) de ≥2 fluxos distintos | Cada fluxo com trecho de código citado — não vale suposição por nome parecido |
| Branch/commit analisado | `git branch --show-current` + `git rev-parse HEAD` | Resultado registrado no relatório antes da análise começar |

Regras gerais de auditoria:
- Toda linha de tabela no relatório final deve ter um caminho de arquivo (e, quando aplicável, número de linha) como evidência.
- Se um sensor não confirmar uma afirmação, ela não entra como fato — vai para "Gaps/Observações" com o motivo.
- Nenhuma regra de negócio, nome de campo ou estrutura pode ser citada de memória/inferência — precisa ter sido lida no código durante esta análise.
- Reforço: contrato de entrada (o que a aplicação expõe) não é objeto de verificação detalhada — o foco dos sensores é consulta de saída e persistência.

## Entregável
O resultado final deve ser um **arquivo HTML único, didático e de fácil leitura**, contendo:
- Sumário/índice navegável no topo (âncoras para cada sigla e domínio)
- Cabeçalho com branch analisada, hash do commit e data da análise
- Organização por sigla (IG3, depois XF8) e, dentro de cada uma, por domínio do respectivo escopo:
  - **IG3**: Saldos → Limites
  - **XF8**: Lançamentos Pendentes → Lançamentos Efetivos → Recebíveis
- Para cada domínio, usar seções claras com:
  - **Estímulo(s)** (tabela resumida: Tipo | Rota/Tópico | Classe/método de entrada)
  - **Consulta (saída)** (tabela: Aplicação | Endpoint consultado | Método | Parâmetros | Headers | Resposta)
  - **Regras de transformação** (tabela: Campo(s) | Regra de negócio | Implementação)
  - **Persistência** (tabela: Tipo | Nome da base | Tipo da base | Nome da tabela | Estrutura | Componente responsável)
  - **Mapeamento campo → coluna** (tabela: Campo origem | Coluna destino | Transformação aplicada)
  - **Tabelas compartilhadas** (se houver), destacada visualmente (ex: cor de alerta) por ser ponto de atenção
  - **Glue Jobs relacionados** (se houver)
  - **Sensores aplicados** (mini-tabela por seção: o que foi verificado e status — confirmado/não verificado)
  - **Gaps/Observações**
- Um diagrama simples (texto estruturado, ASCII-like ou blocos HTML/CSS) representando o fluxo: Estímulo → Consulta (API/Datamesh) → (transformação) → Persistência/Evento → Consumidor, por domínio
- Indicação visual clara (ex: badge verde "verificado" / badge amarelo "não verificado") em cada afirmação que passou ou não pelos sensores
- Estilo visual limpo, tipografia legível, cores para diferenciar sigla/domínio, tabelas com bom contraste — sem exagero visual que atrapalhe leitura técnica
- Seção final com a lista de todos os arquivos/caminhos do repositório usados como fonte

## Instruções de execução
1. Fazer checkout/pull da branch `develop` em cada repositório das aplicações IG3 e XF8, e registrar branch + hash do commit (sensor de branch/commit).
2. Explorar o(s) repositório(s) (controllers, consumers de SQS/Kafka, clients HTTP de saída, glue jobs, migrations/schemas, DDLs, arquivos de infra).
3. Identificar os estímulos relevantes (HTTP inbound, SQS, Kafka, schedule) apenas como ponto de partida — **não detalhar o contrato de entrada da aplicação IG3/XF8 como objetivo da análise**.
4. A partir de cada estímulo, rastrear a consulta de saída (API Produto/Datamesh) e a persistência resultante (direta ou indireta via evento).
5. Respeitar o escopo de domínios por sigla definido acima — não misturar domínios entre siglas.
6. Para cada afirmação (estímulo, endpoint consultado, tabela, evento, glue job, campo, tabela compartilhada), aplicar o sensor correspondente da tabela "Sensores e Validações" antes de registrar como confirmada.
7. Toda regra de transformação deve vir acompanhada da justificativa de negócio, quando identificável no código/comentários/documentação — se não for identificável, declarar "regra de negócio não documentada no código-fonte".
8. Antes de finalizar, cruzar todos os destinos de persistência mapeados para identificar tabelas escritas por mais de um fluxo, mesmo entre domínios diferentes, e validar via sensor de tabela compartilhada.
9. Onde a informação não passar em um sensor ou não estiver disponível no repositório, declare isso explicitamente em "Gaps/Observações" em vez de inferir.
10. Gerar o HTML final conforme especificado em "Entregável", garantindo que fique navegável e compreensível para alguém que não conhece o código a fundo.
