# Prompt para Agente de IA — Diagnóstico de Performance: Jornada de Onboarding (YO4 → Kafka → IG3)

> Copie o conteúdo abaixo (a partir de "## OBJETIVO") como prompt/instrução para um agente de IA com acesso ao(s) repositório(s) de código de YO4 e IG3 (e, se possível, acesso de leitura à AWS — DynamoDB, SQS, Kafka/MSK — para inspeção de estrutura, não de dados sensíveis).

---

## OBJETIVO

Você deve realizar um **diagnóstico técnico completo** da jornada de Onboarding de operadores, que envolve as aplicações **YO4** e **IG3**, com o objetivo de **confirmar, refutar ou complementar** um diagnóstico preliminar já levantado (descrito abaixo), e **identificar quaisquer outros problemas de performance ou design não mencionados**.

**IMPORTANTE — regras estritas:**
- **NÃO proponha soluções, refatorações ou recomendações de código.** Seu único entregável é um diagnóstico factual e estruturado da situação atual.
- Onde não for possível confirmar algo por leitura de código/infra, **declare explicitamente como "não confirmado" ou "requer validação com o time"**, em vez de presumir.
- Toda afirmação técnica no relatório final deve estar amarrada a uma evidência concreta: caminho de arquivo, nome de classe/método, definição de infraestrutura (Terraform/CloudFormation/CDK), configuração, ou métrica observada. Evite "provavelmente" sem indicar o que foi checado.
- Se algo no diagnóstico preliminar abaixo estiver **incorreto** à luz do que você encontrar no código, sinalize a divergência claramente — não assuma que o diagnóstico preliminar está certo.

---

## DIAGNÓSTICO PRELIMINAR (a ser confirmado/refutado/complementado)

### Contexto funcional
Onboarding é a jornada de primeiro acesso do operador à plataforma. YO4 descobre e persiste as contas às quais o operador tem permissão (via API externa), e propaga essa informação para IG3 via Kafka, para que IG3 possa disponibilizar consulta de saldos/limites agregados por agrupamento de contas.

### YO4
- Busca contas permitidas em API externa via múltiplas chamadas, usando virtual threads, com semaphore limitando 100 threads ativas por lote.
- Persiste as contas (criação para conta permitida, remoção para conta revogada) em lotes via JDBC.
- A cada lote de 100 registros salvos no banco, publica os 100 eventos Kafka correspondentes (tópico relacionado a produtos financeiros), até esgotar todos os lotes.
- Publicação é assíncrona em relação à jornada percebida pelo operador.
- Hipótese preliminar: YO4 **não é o gargalo** — produz e publica eventos em ritmo rápido e paralelizado.

### IG3
- Consome eventos Kafka e publica em uma fila SQS **FIFO**, com **Group ID = idUsuario** (hipótese: isso serializa todo o processamento das contas de um mesmo operador, impedindo paralelismo mesmo com múltiplos consumers).
- Duas tabelas DynamoDB envolvidas:
  - **3004**: contas permitidas do operador (espelha o que veio de YO4). Alimenta a tela "Conectar Contas" (agrupamento manual).
  - **3001**: agrupamento conta-usuário (quais contas estão agrupadas para visualização agregada de saldo/limite). PK atual = idAgrupamento (hipótese, a confirmar); existe **1 linha por conta** dentro do mesmo agrupamento.
- Feature de auto-agrupamento: durante o onboarding (mesmo dia), cada conta permitida consumida é automaticamente agrupada na 3001.
- Fluxo de escrita atual na 3001, ao chegar um evento de conta permitida (hipótese a confirmar por código):
  1. Salva a conta na 3004.
  2. Cria lock no Redis pela chave do agrupamento.
  3. Recupera (Query) todas as linhas existentes do agrupamento na 3001.
  4. Adiciona a conta nova ao conjunto em memória.
  5. Deleta todas as linhas existentes do agrupamento na 3001.
  6. Regrava todas as linhas (existentes + nova) na 3001.
  7. Libera o lock.
- Esse mesmo padrão (deletar tudo e regravar tudo) é usado também no fluxo de agrupamento **manual** (operador seleciona/desseleciona conta na tela).
- Existe uma coluna de **versão** no agrupamento, que deve ser **igual em todas as linhas** de um mesmo agrupamento, usada para **controle de concorrência otimista (optimistic locking)**.
- A cada 30 minutos roda um processo de **repermissionamento** para operadores já onboardados, que também pode gerar eventos de conta permitida (ganho de acesso) ou revogada (perda de acesso), reutilizando o mesmo fluxo de escrita.
- Hipótese de causa raiz: para um operador com N contas, o custo total de escrita na 3001 durante o onboarding é O(N²) (a cada nova conta, reescreve-se o agrupamento inteiro, que cresce a cada iteração), resultando em lentidão perceptível ("gota a gota") tanto no auto-agrupamento quanto na disponibilidade da própria 3004 (porque o consumo do Kafka/SQS é o mesmo pipeline serializado).
- Hipótese: por causa do FIFO + lock Redis + padrão de escrita O(N), a tela "Conectar Contas" pode mostrar uma lista **parcial e crescente** de contas permitidas se acessada durante o onboarding (ex.: aparecem 100 contas, dá F5, aparecem 200, etc.), em vez da lista completa.

---

## ESCOPO DO DIAGNÓSTICO A REALIZAR

### 1. Estrutura atual das tabelas DynamoDB (3004 e 3001)
- Partition Key e Sort Key reais de cada tabela (confirmar se 3001 realmente usa `idAgrupamento` como PK sozinha, ou já tem alguma composição).
- Todos os **GSIs e LSIs** existentes em cada tabela — nome, chaves, atributos projetados, e **para que cada um é usado** (buscar no código quem faz Query neles).
- Modo de capacidade (on-demand vs. provisioned) e, se provisioned, RCU/WCU configurados — relevante para saber se throttling está ocorrendo.
- Presença ou ausência de TTL, streams (DynamoDB Streams) habilitados, e o que consome esses streams, se houver.
- Tamanho médio/máximo de item e estimativa de quantos itens existem hoje por agrupamento (se der para consultar de forma agregada/amostral, sem expor dados sensíveis).

### 2. Fluxo de escrita real (código) — YO4
- Confirmar exatamente como funciona a busca de contas na API externa: é paginada, é uma chamada por conta, quantas chamadas são feitas por operador em média/no pior caso.
- Confirmar o uso de virtual threads e do semaphore (tamanho do pool, configuração, se é hardcoded ou parametrizável).
- Confirmar o tamanho real do lote de persistência JDBC e de publicação Kafka (é sempre 100? é configurável?).
- Verificar se existe algum rate limiting, retry, circuit breaker ou backpressure na publicação Kafka.
- Verificar se há alguma métrica/log de tempo total de processamento do onboarding por operador (do início da busca de contas até o último evento publicado).

### 3. Fluxo de consumo real (código) — IG3
- Confirmar a configuração real da fila SQS: é FIFO mesmo, qual o Group ID usado, qual a estratégia de deduplicação (content-based ou message deduplication ID explícito), visibility timeout, DLQ configurada.
- Confirmar quantos consumers/pods estão configurados para essa fila, e se HPA (autoscaling) está configurado — se sim, com base em qual métrica.
- Localizar e transcrever (referenciando arquivo/classe/método) o código exato que implementa o fluxo de escrita na 3001 descrito na hipótese (lock Redis → Query → delete → put), confirmando ou corrigindo cada etapa.
- Confirmar se esse é de fato o mesmo código/serviço usado tanto pelo auto-agrupamento (onboarding) quanto pelo agrupamento manual, ou se há alguma diferença sutil entre os dois caminhos.
- Verificar configuração do lock Redis: biblioteca usada, TTL do lock, estratégia de retry/espera, timeout.
- Verificar se existe alguma validação de "o agrupamento pertence ao operador" e exatamente como ela é feita hoje (Query completa, GetItem, campo específico).
- Verificar como e onde a coluna de versão é lida/escrita, e se **existe algum outro consumidor** (outro serviço, relatório, job batch, endpoint de API) que lê a versão diretamente das linhas de conta na 3001 — isso é crítico para qualquer mudança futura de modelagem.
- Verificar se existe alguma lógica de negócio, validação ou consumidor que dependa da 3001 ter exatamente uma linha por conta (ex.: contagem de linhas usada como "quantidade de contas agrupadas").

### 4. Fluxo de leitura — tela "Conectar Contas" e telas de saldo/limite agregado
- Localizar o endpoint/serviço que alimenta a tela "Conectar Contas" e confirmar que ele lê diretamente da 3004 sem cache intermediário (ou identificar se há cache, e qual o TTL).
- Confirmar se existe alguma forma de o frontend/backend saber que o onboarding "ainda não terminou" para aquele operador (ex.: flag de status, contador esperado vs. atual), ou se de fato não há nenhuma sinalização e a tela simplesmente mostra o que existir no momento da consulta.
- Localizar o endpoint/serviço que lê a 3001 para exibir saldos/limites agregados e confirmar como ele monta a lista de contas do agrupamento (Query, tamanho médio de resultado, paginação).

### 5. Repermissionamento (job de 30 em 30 minutos)
- Localizar o job/scheduler responsável, confirmar sua frequência real e se realmente reutiliza o mesmo pipeline (mesmos tópicos Kafka, mesma fila SQS, mesmo código de escrita) do onboarding.
- Verificar se existe alguma diferença de volume esperado (repermissionamento tende a gerar poucos eventos por execução, versus onboarding que gera todos de uma vez) e se isso já causa algum efeito colateral observável (ex.: filas de onboarding e repermissionamento concorrendo pelos mesmos consumers).

### 6. Observabilidade disponível
- Listar quais métricas/dashboards já existem hoje (CloudWatch, X-Ray, Datadog, New Relic, ou equivalente) que cobrem: idade da mensagem mais antiga na fila SQS (age of oldest message), throttling em DynamoDB, latência de consumo Kafka (consumer lag), tempo de execução do lock Redis, tempo de resposta dos endpoints envolvidos.
- Identificar lacunas de observabilidade — o que **não** existe hoje e impede confirmar quantitativamente o problema (ex.: não há métrica de tempo total de onboarding por operador).

### 7. Pontos não mapeados / novos problemas
- Durante a investigação, identifique e documente **qualquer problema adicional de performance, concorrência, consistência ou design** que não esteja no diagnóstico preliminar acima — mesmo que não pareça a causa principal do "gota a gota" relatado. Exemplos do tipo de coisa a procurar (não exaustivo): outros locks, outras Queries de varredura completa (Scan), N+1 de chamadas a serviços externos, falta de índice para uma consulta frequente, configuração de timeout/retry inadequada, possíveis condições de corrida entre onboarding e repermissionamento rodando simultaneamente para o mesmo operador.

---

## FORMATO DE SAÍDA

Gere um único arquivo Markdown, `diagnostico-onboarding-ig3.md`, com a seguinte estrutura:

```markdown
# Diagnóstico de Performance — Jornada de Onboarding (YO4 → Kafka → IG3)

Data do diagnóstico: <data>
Escopo analisado: <repositórios/branches/ambientes verificados>

## 1. Resumo executivo
(3-5 frases, só fatos confirmados, sem recomendações)

## 2. Estrutura atual das tabelas
### 3004
(PK, SK, GSIs/LSIs, capacidade, streams, TTL)
### 3001
(idem)

## 3. Fluxo YO4 (confirmado por código)
(com referências a arquivo/classe/método)

## 4. Fluxo IG3 — consumo e escrita (confirmado por código)
(com referências a arquivo/classe/método)

## 5. Fluxo de leitura (telas/endpoints)
(com referências)

## 6. Repermissionamento
(com referências)

## 7. Confirmações e divergências em relação ao diagnóstico preliminar
| Item do diagnóstico preliminar | Confirmado / Refutado / Não confirmado | Evidência |
|---|---|---|

## 8. Novos problemas identificados (não previstos no diagnóstico preliminar)
(lista, cada um com evidência e severidade percebida — sem propor solução)

## 9. Lacunas de observabilidade
(o que falta medir para quantificar o problema com precisão)

## 10. Perguntas em aberto / pontos que exigem validação com o time
(qualquer coisa que não pôde ser confirmada só por código/infra)
```

Não inclua seção de recomendações, próximos passos técnicos ou soluções — isso será feito em uma etapa posterior, em separado, a partir deste diagnóstico.
