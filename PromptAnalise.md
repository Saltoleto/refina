# Contexto


Pipeline: evento Kafka (conta permitida do usuário) → aplicação salva na tabela DynamoDB `3004` → a mesma aplicação publica em uma fila **SQS FIFO** agrupada por `MessageGroupId = userId` → a própria aplicação consome essa fila FIFO → grava o agrupamento na tabela DynamoDB `3001` (`tbig3001_agru_agrd_finn`).

**Sintoma:** o processo de onboarding fica muito lento quando o operador tem muitas contas (ex: 1000+). A lentidão cresce de forma aproximadamente linear com o número de contas do usuário.

**Hipótese de causa raiz (a validar):**

1. O FIFO com `groupId = userId` foi usado como mecanismo de lock para serializar o processamento das mensagens de um mesmo usuário, evitando uma race condition na criação do agrupamento.
2. A entidade `AgrupamentoProdutoEntity` tem `HASH_KEY = idAgrupamento` (UUID gerado) e `RANGE_KEY = idProduto`, com um GSI (`INDEX_CODIGO_USUARIO_AGRUPAMENTO_PRODUTO`) para buscar agrupamento por `idUsuario`. Ou seja, cada conta/produto já é um item independente na 3001 (PK+SK distintos) — não deveria haver conflito entre contas diferentes do mesmo usuário.
3. O método `AgrupamentoServiceImpl.salvarAgrupamentoContaLogada` faz um **check-then-act**:
   ```java
   if (agrupamentoRepository.usuarioPossuiAgrupamento(agrupamento.getIdUsuario())) {
       throw new AgrupamentoJaCriadoException(agrupamento.getIdUsuario());
   }
   return agrupamentoRepository.save(agrupamento);
   ```
   Essa checagem consulta o GSI por `idUsuario` e, se não encontrar, salva um novo agrupamento com `idAgrupamento` aleatório. Entre o `if` e o `save`, duas mensagens concorrentes do mesmo usuário podem não encontrar nada e criar dois agrupamentos distintos — essa é a race real que o FIFO estaria evitando, e ela só importa na criação do **primeiro** agrupamento do usuário, não nas gravações subsequentes de cada conta.
4. Como consequência, a aplicação está usando ordenação de fila (que serializa TODAS as N mensagens de um usuário, uma a uma) para proteger uma janela de concorrência que na prática só existe uma vez por onboarding — o que faz o tempo total escalar linearmente com o número de contas.

**Proposta de solução discutida (a avaliar/validar):**
Tornar `idAgrupamento` determinístico a partir do `idUsuario` (ex: `UUID.nameUUIDFromBytes(idUsuario.getBytes())`) e substituir o check-then-act por um `PutItem` condicional (`attribute_not_exists`) no DynamoDB, capturando `ConditionalCheckFailedException` como sinal de que o agrupamento já existe. Isso elimina a necessidade de query prévia e, consequentemente, a necessidade do FIFO/serialização por usuário — permitindo processar as N contas de um onboarding em paralelo.

# Objetivo da análise

1. **Validar o comportamento atual do código** relacionado à gestão do agrupamento na tabela 3001, especificamente:
   - Confirmar (ou refutar) que a criação do agrupamento segue o padrão check-then-act descrito acima.
   - Mapear todos os pontos do código que leem, criam ou atualizam registros na 3001 relacionados a um mesmo `idUsuario` (incluindo fluxos de adição de conta a um agrupamento já existente, atualização de dados de produto, remoção de conta, etc.), e identificar se existe alguma outra race condition além da já mapeada.
   - Verificar se `idAgrupamento` é exposto/consumido como identificador externo por outros sistemas, APIs ou eventos publicados (`emitirEventoAgrupamentoCriado`, `solicitaAtualizacaoProduto`, etc.), o que impactaria a viabilidade de trocar sua geração para determinística.
   - Verificar se existe algum outro motivo (não relacionado à race condition) para o uso de SQS FIFO com groupId=userId nesse fluxo (ex: ordem de eventos de negócio relevante, idempotência de outro tipo, requisito de auditoria).

2. **Após validar o comportamento atual**, apresentar as possibilidades concretas para eliminar a lentidão no onboarding de usuários com muitas contas, avaliando ao menos:
   - Tornar `idAgrupamento` determinístico + `PutItem` condicional, removendo a necessidade do FIFO.
   - Manter FIFO, porém eliminar o gargalo via tabela/registro de controle separado (find-or-create isolado) protegendo apenas a criação do agrupamento, liberando o restante do processamento para paralelismo (standard queue ou groupId mais granular).
   - Uso de optimistic locking (atributo de versão + `ConditionExpression`) como alternativa ao lock por fila.
   - Qualquer outra abordagem que a análise do código real revele ser mais adequada, considerando os pontos de integração existentes (eventos emitidos, cache, outros consumidores da 3001).

Para cada opção sugerida, indicar impacto em compatibilidade (contratos de API/eventos existentes), esforço estimado de implementação e riscos.
