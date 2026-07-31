# Tarefa: Mapeamento de Origem e Persistência de Dados — IG3 e XF8
## Execução em fases (para otimizar uso de contexto/tokens)

Esta tarefa deve ser executada em 3 fases sequenciais. Cada fase gera um artefato intermediário que serve de entrada para a próxima — não é necessário manter todo o contexto bruto da investigação até o final.

## Contexto
As aplicações das siglas IG3 e XF8 NÃO são Golden Source. Todos os dados são consultados em APIs Produto e/ou Datamesh, e depois persistidos em bases de dados próprias de cada sigla. A persistência pode ser:
(a) direta, no mesmo fluxo síncrono da consulta, ou
(b) indireta, quando o resultado da consulta vira um evento (SQS/SNS/Kafka) consumido por outro componente que faz a persistência.

Além disso, existem Glue Jobs que consultam Datamesh e persistem dados nas bases da sigla, fora do fluxo das APIs.

## Importante: o que NÃO é objetivo desta análise
**Não é objetivo mapear os endpoints/contratos que as aplicações IG3 e XF8 expõem** (a API de entrada delas). O objeto do mapeamento é o que a aplicação **consulta** (em APIs Produto/Datamesh) e onde **persiste** o resultado.

## Conceito de "estímulo" (ponto de partida do fluxo)
Cada fluxo começa a partir de um estímulo que aciona a aplicação: chamada HTTP recebida, consumo SQS, consumo Kafka, trigger agendado (schedule/cron), ou outro evento assíncrono (SNS, etc.). O estímulo deve ser identificado apenas como ponto de entrada do fluxo (para saber onde a investigação começa) — documentar de forma resumida (tipo, rota/tópico, classe/método), sem detalhar seu contrato de entrada. O detalhamento completo é exigido para a consulta de saída e para a persistência, não para o estímulo.

## Escopo por sigla

**IG3**
- Saldos
- Limites de contas

**XF8**
- Lançamentos Pendentes
- Lançamentos Efetivos
- Recebíveis

---

## FASE 0 — Setup (executar uma única vez)

1. Fazer checkout/pull da branch `develop` em cada repositório das aplicações IG3 e XF8 (não usar `main`/`master` nem branches de feature).
2. Registrar branch confirmada (`git branch --show-current`) e hash do commit (`git rev-parse HEAD`) de cada repositório.
3. Listar (sem abrir conteúdo ainda) diretórios/arquivos candidatos por padrão de nome: `*Controller`, `*Consumer`, `*Listener`, `*Client`, `*Repository`, `*Job`, `*glue*`, `*.sql`, migrations/, terraform/, cdk/.
4. Salvar essa lista como inventário inicial — as fases seguintes só devem abrir arquivos que apareçam nesse inventário ou que sejam referenciados diretamente por eles.

**Saída da Fase 0:** inventário de arquivos candidatos + branch/commit registrados.

---

## FASE 1 — Investigação e extração (subtarefa separada por sigla + domínio)

Executar uma subtarefa por combinação sigla/domínio (6 no total):
- IG3: Saldos | IG3: Limites
- XF8: Lançamentos Pendentes | XF8: Lançamentos Efetivos | XF8: Recebíveis

Cada subtarefa, usando apenas o inventário da Fase 0 como ponto de partida (evitar `view` de diretórios inteiros; usar `grep`/busca direcionada), deve levantar:

### 1. Estímulo
Tipo (HTTP/SQS/Kafka/schedule), rota/tópico, classe/método de entrada. Resumido, sem detalhar contrato.

### 2. Consulta de saída (API Produto / Datamesh)
- Aplicação responsável
- Endpoint completo (host + path) e verbo HTTP **da API consultada**
- Parâmetros (path, query, body)
- Headers enviados
- Estrutura do payload de resposta
- Client/biblioteca usado e onde está definido no código

### 3. Regras de transformação
- Regra de negócio em linguagem clara (o quê e por quê)
- Trecho/classe/método onde está implementada
- Dependência de configuração externa (feature flag, etc.), se houver
- Se não houver transformação: declarar "sem transformação — persistência 1:1"

### 4. Persistência (direta ou via evento)
- Nome da base de dados, tipo da base, nome da tabela/coleção
- Estrutura completa (colunas, tipos, chaves, índices relevantes)
- Mapeamento campo de origem → coluna, refletindo a transformação aplicada
- Camada/classe responsável (repository, DAO, etc.)
- Se via evento: tópico/fila (SNS/SQS/Kafka), estrutura do payload, consumidor responsável (sinalizar "não encontrado neste repo" se aplicável)

### 5. Glue Jobs (quando aplicável ao domínio)
- Nome do job e trigger
- Query/fonte no Datamesh (tabela, database, formato)
- Regras de transformação aplicadas
- Base/tipo/tabela de destino e estrutura completa

Aplicar os sensores da tabela "Sensores e Validações" a cada afirmação antes de registrá-la como confirmada.

**Saída de cada subtarefa:** bloco estruturado (JSON/markdown), sem prosa nem HTML:

```json
{
  "sigla": "IG3",
  "dominio": "Saldos",
  "estimulos": [...],
  "consultas": [...],
  "transformacoes": [...],
  "persistencias": [...],
  "mapeamento_campos": [...],
  "glue_jobs": [...],
  "sensores": [...],
  "gaps": [...],
  "arquivos_fonte": [...]
}
```

---

## FASE 2 — Consolidação e cruzamento (uma execução, só com saídas da Fase 1)

Usando apenas os 6 blocos estruturados (não reabrir código-fonte, exceto para resolver inconsistência pontual):

1. Cruzar todas as `persistencias` de todos os domínios/siglas para identificar **tabelas escritas por mais de um fluxo** (tabelas compartilhadas).
2. Para cada tabela compartilhada, aplicar o sensor de evidência de escrita em ≥2 fluxos, listando: fluxos que escrevem, colunas por fluxo, tratamento de conflito (ex: upsert por chave, timestamp).
3. Consolidar todos os `gaps`/sensores não confirmados em uma lista única.
4. Consolidar lista final de `arquivos_fonte` (deduplicada).

**Saída da Fase 2:** bloco estruturado consolidado, pronto para virar HTML.

---

## FASE 3 — Geração do HTML (uma execução, apenas formatação)

Usando apenas o bloco consolidado da Fase 2 (não reabrir código-fonte), gerar **um único arquivo HTML**, didático e navegável, contendo:

- Sumário/índice navegável no topo (âncoras por sigla e domínio)
- Cabeçalho com branch, hash do commit e data da análise
- Organização: **IG3** (Saldos → Limites), depois **XF8** (Lançamentos Pendentes → Lançamentos Efetivos → Recebíveis)
- Por domínio: Estímulo | Consulta (saída) | Regras de transformação | Persistência | Mapeamento campo→coluna | Tabelas compartilhadas (destacadas visualmente, ex: cor de alerta) | Glue Jobs | Sensores aplicados (badge verde "verificado" / amarelo "não verificado") | Gaps/Observações
- Diagrama simples (HTML/CSS ou texto estruturado): Estímulo → Consulta (API/Datamesh) → Transformação → Persistência/Evento → Consumidor, por domínio
- Estilo visual limpo, tipografia legível, cores diferenciando sigla/domínio, boas tabelas com contraste — sem exagero visual
- Seção final com lista de todos os arquivos/caminhos usados como fonte

Esta fase é puramente de formatação — não deve re-executar buscas no repositório.

---

## Sensores e Validações

| Tipo de afirmação | Sensor / comando de verificação | Critério de aprovação |
|---|---|---|
| Estímulo (rota/tópico de entrada) | `grep -r` pela rota/tópico/nome do listener no código | String encontrada, com caminho registrado |
| Endpoint consultado (API Produto/Datamesh) | `grep -r` pelo path literal no client/config | String encontrada, com caminho registrado |
| Nome de tabela | Busca no DDL/migration/schema (Flyway, Liquibase, `.sql`) | Definição encontrada, não só referência em query/repository |
| Estrutura da tabela | Extração direta do DDL/migration | Bate com entity mapeada; divergência é registrada |
| Tópico/fila (SNS/SQS/Kafka) | Busca em config/infra (application.yml, Terraform, CDK, CloudFormation) | Nome confirmado fora do código de aplicação |
| Glue Job | Busca em infra (Terraform/CDK) ou export de configuração do job | Script/job encontrado com fonte/destino explícitos |
| Campo mapeado | Existe na resposta da API/evento E na estrutura da tabela | Ambos confirmados; senão, inconsistência registrada |
| Tabela compartilhada | Evidência de escrita (insert/update/save) de ≥2 fluxos distintos | Cada fluxo com trecho de código citado |
| Branch/commit analisado | `git branch --show-current` + `git rev-parse HEAD` | Registrado antes da Fase 1 |

**Regra geral:** se um sensor não confirmar uma afirmação, ela não entra como fato — vai para `gaps`/"Gaps/Observações" com o motivo. Nenhuma regra de negócio, nome de campo ou estrutura pode ser citada de memória/inferência — precisa ter sido lida no código durante esta análise. Contrato de entrada (o que a aplicação expõe) não é objeto de verificação detalhada.

---

## Instruções gerais de execução
1. Executar as fases em ordem (0 → 1 → 2 → 3); cada fase só usa a saída da anterior, não o repositório bruto (exceto Fase 1, que acessa o código).
2. Respeitar o escopo de domínios por sigla — não misturar domínios entre siglas.
3. Não presumir estrutura de payload, nome de tabela ou regra de negócio — extrair diretamente do código.
4. Toda regra de transformação deve vir com justificativa de negócio, quando identificável — senão, declarar "regra de negócio não documentada no código-fonte".
5. Onde a informação não passar em um sensor ou não estiver disponível, declarar isso explicitamente em vez de inferir.
6. Entregável final: o arquivo HTML da Fase 3.
