---
name: firebird-master
description: "Use sempre que precisar analisar, otimizar, debugar ou criar comandos SQL, DDL, DML, procedures e triggers para banco de dados Firebird (versões 2.5, 3.0, 4.0 e 5.0). Acionar para análise de plano de execução (PLAN), identificação de NATURAL SCAN, criação e revisão de índices, diagnóstico de transações longas (gap OAT/OIT e Garbage Collection), boas práticas de PSQL e integração com Delphi FireDAC/IBX."
---

# Firebird Master — Especialista em Otimização, Arquitetura e SQL Multi-Versão

Esta skill define o conhecimento aprofundado, os padrões de raciocínio e as heurísticas de otimização para banco de dados **Firebird**, cobrindo desde o legado (**Firebird 2.5**) até as versões modernas (**Firebird 3.0, 4.0 e 5.0**).

O agente atua como um DBA e Engenheiro de Banco de Dados Firebird experiente, focando em:
1. **Análise de Consultas e Planos de Execução (`PLAN`)** — eliminação de gargalos e leituras não indexadas (`NATURAL`).
2. **Estratégia de Índices** — seletividade, índices compostos, índices por expressão e limitações de versão.
3. **Matriz de Compatibilidade Cross-Version** — respeito estrito às capacidades de cada versão do engine.
4. **Arquitetura MVCC e Transações** — controle do gap OAT/OIT, prevenção de inchaço do `.fdb` e boas práticas com Delphi FireDAC.
5. **Integração com Ferramentas MCP** — acionamento automático de inspeção ativa e medição de telemetria quando o MCP Server Firebird estiver disponível.

---

## 1. Matriz de Compatibilidade e Recursos por Versão

Antes de sugerir qualquer sintaxe, tipo de dado ou melhoria, **sempre confirme ou identifique a versão do Firebird em uso**. Caso não informada, assuma a versão padrão do projeto (ou questione o usuário).

| Recurso / Sintaxe | Firebird 2.5 (Legado) | Firebird 3.0 | Firebird 4.0 | Firebird 5.0 |
| :--- | :--- | :--- | :--- | :--- |
| **Tipo `BOOLEAN`** | ❌ Não nativo (usar `CHAR(1)` / `SMALLINT`) | ✅ Nativo (`TRUE`, `FALSE`, `UNKNOWN`) | ✅ Nativo | ✅ Nativo |
| **Window Functions (`OVER`)** | ❌ Não suportado | ✅ `ROW_NUMBER()`, `RANK()`, `DENSE_RANK()` | ✅ Completo | ✅ Completo |
| **Packages (`PACKAGE`)** | ❌ Não suportado | ✅ `CREATE PACKAGE` / `BODY` | ✅ Completo | ✅ Completo |
| **Funções em SQL (`CREATE FUNCTION`)** | ❌ Apenas UDFs externas (C/C++) | ✅ Nativo em PSQL | ✅ Nativo em PSQL | ✅ Nativo em PSQL |
| **Timezones** | ❌ Apenas `TIMESTAMP` sem fuso | ❌ Apenas `TIMESTAMP` sem fuso | ✅ `TIMESTAMP WITH TIME ZONE` | ✅ `TIMESTAMP WITH TIME ZONE` |
| **Tipos de Alta Precisão** | ❌ Max `NUMERIC(18,x)` / `BIGINT` | ❌ Max `NUMERIC(18,x)` / `BIGINT` | ✅ `INT128`, `DECFLOAT(16/34)` | ✅ `INT128`, `DECFLOAT(16/34)` |
| **Timeout de Query** | ❌ Não nativo | ❌ Não nativo | ✅ `SET STATEMENT TIMEOUT` | ✅ `SET STATEMENT TIMEOUT` |
| **Modo de Transação Moderno** | `READ COMMITTED [RECORD_VERSION]` | `READ COMMITTED [RECORD_VERSION]` | ✅ `READ COMMITTED READ CONSISTENCY` | ✅ `READ COMMITTED READ CONSISTENCY` |
| **Paginação de Queries** | `ROWS X TO Y` ou `FIRST X SKIP Y` | `OFFSET X ROWS FETCH NEXT Y ROWS ONLY` | `OFFSET / FETCH` padrão SQL | `OFFSET / FETCH` padrão SQL |
| **Execução Paralela** | ❌ Monothread por query | ❌ Monothread por query | ❌ Monothread por query | ✅ `ParallelWorkers` (Varredura e Índices) |
| **Batch Processing na API** | ❌ Um comando por roundtrip | ❌ Um comando por roundtrip | ❌ Um comando por roundtrip | ✅ Execução nativa em lote |
| **Profiler Integrado** | ❌ Depende de trace externo | ❌ Trace API | ❌ Trace API | ✅ `RDB$PROFILER` nativo |
| **Limite de Tamanho de Chave de Índice** | ⚠️ Máximo **1/4 do Page Size** (~1KB em 4KB) | Ampliado e comprimido | Ampliado com index key compression | Ampliado com index key compression |

### ⚠️ Armadilhas Críticas de Versão:
1. **Firebird 2.5 — Estouro de Chave de Índice**:
   - Em FB 2.5 com página de 4096 bytes, uma chave de índice composta com campos `VARCHAR(100) CHARACTER SET UTF8` (4 bytes por char = 400 bytes) mais outros campos pode estourar o limite de ~1024 bytes (`key size exceeds implementation restriction`).
   - *Solução:* Aumentar o Page Size para 8192 ou 16384 bytes, ou criar índice por Hash / expressão seletiva.
2. **Firebird 3.0+ — Subconsultas e Otimizador de Custos**:
   - O FB 3.0 introduziu um novo otimizador baseado em custos. Queries que usavam `+0` ou truques de forçar plano no FB 2.5 podem se comportar de forma diferente. Nunca use hints manuais de `PLAN` sem validar o custo real.
3. **Firebird 4.0+ — Isolamento `READ CONSISTENCY`**:
   - O padrão histórico `READ COMMITTED RECORD_VERSION` apresentava comportamentos onde updates concorrentes podiam ver versões mistas. O FB 4.0 introduziu `READ CONSISTENCY` como padrão recomendado no `firebird.conf`, garantindo uma fotografia consistente da declaração sem locks fantasmas.

---

## 2. Análise Cirúrgica do Plano de Execução (`PLAN`)

O plano de execução (`PLAN`) é o mapa gerado pelo otimizador do Firebird para responder à consulta.

### Anatomia dos Operadores do PLAN:
- `PLAN (TABELA NATURAL)`:
  - **O que significa:** O banco lê a tabela linha por linha, do primeiro ao último registro alocado, sem usar nenhum índice.
  - **Quando é aceitável:** Em tabelas pequenas (< 1.000 registros), tabelas de parâmetros do sistema ou quando a query realmente precisa retornar mais de 70% de todos os registros da tabela.
  - **Quando é ALERTA VERMELHO:** Em tabelas transacionais (itens de venda, notas fiscais, lançamentos financeiros, logs). Se a tabela tiver 500.000 linhas, uma leitura `NATURAL` fará 500.000 leituras de página/registro a cada execução.
- `PLAN (TABELA INDEX (IDX_NOME))`：
  - **O que significa:** O banco navega na árvore B do índice `IDX_NOME` e acessa diretamente os ponteiros dos registros correspondentes.
- `PLAN (TABELA ORDER IDX_NOME)`:
  - **O que significa:** O banco lê a tabela na ordem física das chaves do índice, dispensando uma etapa posterior de ordenação em memória ou disco (`SORT`).
- `PLAN SORT (...)`:
  - **O que significa:** O resultado foi armazenado em memória temporária (ou arquivo `.tmp` do Firebird) para sofrer ordenação (`ORDER BY`, `GROUP BY`, `DISTINCT`).
  - **Otimização:** Se o volume for alto, verifique se um índice composto cobrindo o filtro `WHERE` e a coluna do `ORDER BY` elimina a necessidade do `PLAN SORT`.
- `PLAN JOIN (...)` vs `PLAN MERGE (...)` vs `PLAN HASH (...)`:
  - `JOIN`: Nested Loop Join tradicional (ideal quando a tabela externa é pequena e a interna tem índice direto).
  - `MERGE`: Sort Merge Join (usado quando ambas as tabelas estão ordenadas por índice na condição de junção).
  - `HASH` (FB 3.0+ e aprimorado no 5.0): Cria tabela hash em memória para cruzar grandes conjuntos sem índices ótimos.

---

## 3. Catálogo de Anti-Padrões de Query e Reescrita

### Anti-Padrão 1: Função no Predicado `WHERE` (Anulação de Índice)
- ❌ **Código Ineficiente (`NATURAL` garantido):**
  ```sql
  SELECT ID, VALOR, DATA_EMISSAO 
  FROM VENDAS 
  WHERE EXTRACT(YEAR FROM DATA_EMISSAO) = 2026;
  ```
- ✅ **Reescrita Otimizada (`PLAN INDEX`):**
  ```sql
  SELECT ID, VALOR, DATA_EMISSAO 
  FROM VENDAS 
  WHERE DATA_EMISSAO >= '2026-01-01' AND DATA_EMISSAO < '2027-01-01';
  ```

### Anti-Padrão 2: `UPPER()` sem Índice por Expressão
- ❌ **Código Ineficiente:**
  ```sql
  SELECT ID, NOME FROM CLIENTES WHERE UPPER(NOME) = 'LUCAS GABRIEL';
  ```
  *(O índice padrão em `NOME` não é utilizado porque a função `UPPER` transforma o valor).*
- ✅ **Solução Firebird (Índice por Expressão):**
  ```sql
  CREATE INDEX IDX_CLIENTES_NOME_UPPER ON CLIENTES COMPUTED BY (UPPER(NOME));
  ```
  *Nota:* No Firebird, o predicado deve bater exatamente com a expressão do índice: `WHERE UPPER(NOME) = '...'`.

### Anti-Padrão 3: Curinga no Início de `LIKE` (`%TERMO%`)
- ❌ **Problema:** `WHERE NOME LIKE '%SILVA%'` obriga o Firebird a fazer varredura `NATURAL`, pois árvores B só buscam eficientemente por prefixo (`'SILVA%'`).
- ✅ **Mitigação:**
  1. Se a busca por prefixo for viável para o negócio, use `WHERE NOME LIKE 'SILVA%'` (utiliza índice).
  2. Se a busca por substring for indispensável, combine com outros filtros seletivos indexados (ex: `WHERE STATUS = 'A' AND DATA_CADASTRO >= :DATA AND NOME LIKE '%SILVA%'`), permitindo que o índice filtre primeiro por `STATUS` e `DATA_CADASTRO`.

### Anti-Padrão 4: `IN (SELECT ...)` Ineficiente vs `EXISTS` ou `INNER JOIN`
- ❌ **Código Ineficiente (Especialmente no FB 2.5):**
  ```sql
  SELECT P.ID, P.DESCRICAO 
  FROM PRODUTOS P
  WHERE P.ID IN (SELECT I.ID_PRODUTO FROM PEDIDOS_ITENS I WHERE I.DATA_VENDA >= '2026-01-01');
  ```
- ✅ **Reescrita Recomendada:**
  ```sql
  SELECT P.ID, P.DESCRICAO 
  FROM PRODUTOS P
  WHERE EXISTS (
    SELECT 1 
    FROM PEDIDOS_ITENS I 
    WHERE I.ID_PRODUTO = P.ID 
      AND I.DATA_VENDA >= '2026-01-01'
  );
  ```

### Anti-Padrão 5: `SELECT *` com Colunas `BLOB`
- ❌ **Problema:** `SELECT *` carrega ponteiros de BLOB e páginas de armazenamento secundário mesmo quando a tela só precisa exibir colunas textuais simples.
- ✅ **Regra:** Liste explicitamente as colunas necessárias e traga campos BLOB (`BLOB SUB_TYPE TEXT` ou `BINARY`) apenas sob demanda.

---

## 4. Regras de Ouro para Índices no Firebird

1. **Seletividade de Índice (`RDB$STATISTICS`)**:
   - A seletividade varia de `0.0` a `1.0`. Quanto **menor** o valor (mais próximo de 0), **mais seletivo** é o índice (melhor).
   - Um índice em um campo com 2 valores possíveis (ex: `ATIVO` = 'S'/'N') tem seletividade ~0.5 (péssima). O otimizador frequentemente ignorará esse índice e fará `NATURAL`.
   - **Nunca crie índice isolado em campos booleanos ou de status com baixa cardinalidade**.
2. **Índices Compostos e a Regra do Prefixo**:
   - Um índice em `(EMPRESA, DATA_EMISSAO, STATUS)` só pode ser usado se a consulta filtrar por `EMPRESA`.
   - Se a query filtrar apenas por `DATA_EMISSAO`, esse índice composto **NÃO será utilizado**!
3. **Chaves Estrangeiras (FKs)**:
   - Toda chave estrangeira deve possuir um índice explícito na tabela filha caso o Firebird não o crie automaticamente com seletividade ideal. Sem índice na coluna de FK, qualquer exclusão ou alteração na tabela pai causará varredura `NATURAL` completa na tabela filha para verificar integridade referencial.
4. **Manutenção de Índices (`SET STATISTICS`)**:
   - Após grandes cargas de dados (`INSERT` / `UPDATE` em massa), as estatísticas dos índices ficam desatualizadas.
   - Script de manutenção periódica:
     ```sql
     SET STATISTICS INDEX IDX_NOME;
     -- Ou para todos os índices do banco:
     EXECUTE BLOCK AS
       DECLARE VARIABLE S VARCHAR(31);
     BEGIN
       FOR SELECT RDB$INDEX_NAME FROM RDB$INDICES WHERE RDB$SYSTEM_FLAG = 0 INTO :S DO
         EXECUTE STATEMENT 'SET STATISTICS INDEX ' || :S;
     END;
     ```

---

## 5. Arquitetura MVCC, Transações e o "Inchaço" do `.fdb`

O Firebird utiliza controle de concorrência por multiversão (MVCC). Sempre que um registro é atualizado (`UPDATE`) ou excluído (`DELETE`), o Firebird não sobrescreve o dado original: ele grava uma **nova versão de registro** (back-version) e encadeia um ponteiro para a versão anterior.

### O Gap OAT / OIT e a Parada do Garbage Collection:
- **OIT (Oldest Interesting Transaction):** Transação mais antiga que ainda não teve suas versões completamente recolhidas.
- **OAT (Oldest Active Transaction):** Transação ativa mais antiga que ainda **não fez COMMIT ou ROLLBACK**.
- **Next Transaction:** Número da próxima transação a ser gerada.
- **A Regra Crítica:**
  > O mecanismo de Coleta de Lixo (Garbage Collection) do Firebird **NÃO PODE** limpar nenhuma versão de registro criada após a transação mais antiga ainda ativa (**OAT**)!
  
Se uma aplicação (ex: tela de PDV, relatório aberto ou sessão no Delphi) abrir uma transação e esquecê-la aberta por horas:
1. O gap entre `OAT` e `Next Transaction` cresce para dezenas de milhares.
2. Nenhuma página de dados pode ser limpa.
3. As consultas subsequentes são forçadas a navegar por cadeias gigantescas de versões antigas de registros.
4. O arquivo `.fdb` cresce descontroladamente e as consultas sofrem degradação severa de I/O.

### Boas Práticas Obrigatórias no Delphi (FireDAC):
- **Transações Curtas e Explícitas:**
  ```pascal
  FDConnection.StartTransaction;
  try
    FDQuery.ExecSQL;
    FDConnection.Commit;
  except
    FDConnection.Rollback;
    raise;
  end;
  ```
- **Proibição de `CommitRetaining`:**
  - `CommitRetaining` persiste as alterações no disco, mas **NÃO encerra o contexto da transação** na engine do Firebird! Ele mantém o número da transação aberto, prendendo o OAT. **Use sempre `Commit` puro**.
- **Configuração de Leitura de Relatórios:**
  - Use `ReadOnly := True` e transação em modo `Read Committed` (com `Wait` e `ReadConsistency` no FB 4+).
  - Feche o dataset imediatamente após carregar os dados ou use paginação `FetchOptions.Mode := fmOnDemand`.

---

## 6. Procedimento de Diagnóstico de Comandos SQL

Quando o usuário apresentar uma query, procedure ou problema de performance, siga rigorosamente este roteiro:

1. **Passo 1: Contexto e Versão**
   - Identifique a versão do Firebird (2.5, 3.0, 4.0 ou 5.0).
2. **Passo 2: Análise Estática do SQL**
   - Inspecione a cláusula `WHERE`, os `JOINs` e o `ORDER BY`.
   - Procure imediatamente pelos anti-padrões catalogados (funções no WHERE, LIKE com %, IN desnecessário, joins sem índices).
3. **Passo 3: Análise do Plano (`PLAN`)**
   - Se o usuário forneceu o plano ou se o MCP Server estiver ativo, analise a saída.
   - Sinalize qualquer ocorrência de `NATURAL` em tabelas não triviais.
4. **Passo 4: Proposta Cirúrgica de Otimização**
   - Apresente a query reescrita com o diff funcional.
   - Apresente os comandos DDL necessários (ex: `CREATE INDEX ...`).
   - Explique a justificativa técnica: por que essa mudança reduz leituras de página e melhora o tempo de resposta.
5. **Passo 5: Alerta de Transações e Volume**
   - Caso o comando envolva atualização massiva (`UPDATE` / `DELETE` em milhares de registros), recomende execução em lotes (`EXECUTE BLOCK` com commit intermediário) para evitar lock contention e saturação da tabela de transações.

---

## 7. Integração com MCP Server (`firebird-mcp`)

Quando o servidor MCP de Firebird estiver configurado e ativo no ambiente, o agente deve priorizar o uso das ferramentas MCP para validar dados reais:

- **Gerenciamento Multi-Bancos:**
  - `fb_list_databases`: Lista todos os bancos de dados configurados no projeto atual (`.firebird-mcp.json`).
  - `fb_use_database(alias)`: Alterna o banco de dados padrão ativo.
  - Todas as ferramentas aceitam o parâmetro opcional `db_alias` para operar diretamente em um banco específico (ex: `fb_query(sql=..., db_alias="auditoria")`).
- **Inspeção e Metadados:**
  - `fb_list_tables` / `fb_list_views` / `fb_list_procedures` / `fb_list_generators`: Listagem rápida de objetos.
  - `fb_inspect_table(table_name, db_alias)`: Inspeciona colunas, domínios, chaves primárias/estrangeiras e seletividade de índices (`RDB$STATISTICS`).
- **Performance e Análise:**
  - `fb_explain_plan(sql, db_alias)`: Extrai o plano de execução compilado (`PLAN`), detecta leituras sequenciais (`NATURAL SCAN`) e orienta sobre índices.
  - `fb_database_health(db_alias)`: Diagnostica a saúde do banco via `MON$` (ODS, page size e gap entre OAT e Next Transaction).
- **Execução Segura:**
  - `fb_query(sql, limit, db_alias)`: Executa consultas `SELECT` estritamente de leitura com limite de linhas e commit imediato.
  - `fb_execute_dml(sql, db_alias)`: Executa comandos `INSERT`, `UPDATE`, `DELETE`.
  - `fb_execute_ddl(ddl, db_alias)`: Executa comandos estruturais `CREATE INDEX`, `ALTER TABLE`, etc.

Caso o MCP não esteja conectado ou o ambiente seja puramente de código/Delphi offline, **execute a análise estática completa com as heurísticas desta skill**.
