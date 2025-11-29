# 📚 Data Ingestion com Lakeflow Connect - Guia Didático

## 🎯 O que é este projeto?

Este projeto ensina como **trazer dados de arquivos CSV para dentro do Databricks** de forma organizada e profissional, usando as melhores práticas de engenharia de dados.

Imagine que você tem dados de sinistros de seguros em arquivos CSV e precisa:
- Armazenar esses dados de forma segura
- Processar e limpar os dados
- Disponibilizar para análises e relatórios

Este projeto mostra **4 formas diferentes** de fazer isso!

---

## 🏗️ Conceitos Fundamentais

### 1. Unity Catalog - O "Organizador" dos seus dados

Pense no Unity Catalog como um **sistema de pastas super organizado**:

```
Unity Catalog
└── Catálogo (smart_claims_dev)          ← Como uma "pasta principal"
    ├── Schema 00_landing                 ← Subpasta para dados brutos
    ├── Schema 01_bronze                  ← Subpasta para dados originais
    ├── Schema 02_silver                  ← Subpasta para dados limpos
    └── Schema 03_gold                    ← Subpasta para dados prontos
```

**Por que usar?**
- Organização clara dos dados
- Controle de quem pode acessar o quê
- Facilita encontrar as informações

### 2. Arquitetura Medallion - As "3 Camadas de Qualidade"

É como processar café:
- **🥉 Bronze (Landing)**: Grãos crus (dados como chegaram)
- **🥈 Silver**: Grãos torrados (dados limpos e validados)
- **🥇 Gold**: Café pronto (dados agregados para consumo)

**Benefícios:**
- Sempre tem os dados originais guardados (Bronze)
- Pode refazer o processamento se algo der errado
- Dados ficam cada vez mais refinados

### 3. Volumes - O "HD Externo" do Databricks

Volumes são espaços para guardar **arquivos** (CSV, JSON, imagens, etc):

```sql
CREATE VOLUME smart_claims_dev.00_landing.sql_server
```

**Tradução:** "Crie um espaço de armazenamento chamado 'sql_server' dentro da pasta '00_landing'"

**Onde usar:** `/Volumes/smart_claims_dev/00_landing/sql_server/claims.csv`

---

## 📖 Explicação dos Notebooks

### Notebook 1: Criação de Catálogos e Schemas

**O que faz:** Cria a estrutura organizacional básica

#### Comando 1: Criar o Catálogo
```sql
CREATE CATALOG IF NOT EXISTS smart_claims_dev
COMMENT 'Catálogo principal para o projeto Smart Claims'
```

**Explicação linha por linha:**
- `CREATE CATALOG` = "Crie uma pasta principal"
- `IF NOT EXISTS` = "Só crie se ainda não existir" (evita erros)
- `smart_claims_dev` = Nome do catálogo
- `COMMENT` = Descrição para documentar

#### Comando 2: Criar os Schemas (Camadas)
```sql
CREATE SCHEMA IF NOT EXISTS smart_claims_dev.00_landing
COMMENT 'Zona de landing - recepção de dados brutos'
```

**Explicação:**
- `CREATE SCHEMA` = "Crie uma subpasta"
- `smart_claims_dev.00_landing` = "Dentro do catálogo smart_claims_dev, crie a pasta 00_landing"
- Repete para: `01_bronze`, `02_silver`, `03_gold`

#### Comando 3: Verificar o que foi criado
```sql
SHOW SCHEMAS IN smart_claims_dev
```

**Resultado esperado:**
```
00_landing
01_bronze
02_silver
03_gold
```

---

### Notebook 2: Criação de Volumes

**O que faz:** Cria espaços para armazenar arquivos

#### Comando: Criar Volume
```sql
CREATE VOLUME IF NOT EXISTS smart_claims_dev.00_landing.sql_server
COMMENT 'Volume para armazenar dados do SQL Server'
```

**Explicação:**
- Cria um "HD externo" chamado `sql_server`
- Localização: dentro do schema `00_landing`
- Aqui você vai colocar os arquivos CSV

**Como usar depois:**
```sql
-- Listar arquivos no volume
LIST '/Volumes/smart_claims_dev/00_landing/sql_server'

-- Ler um arquivo
SELECT * FROM csv.`/Volumes/smart_claims_dev/00_landing/sql_server/claims.csv`
```

---

### Notebook 3: Data Ingestion (Ingestão de Dados)

**O que faz:** Mostra 3 formas de trazer dados do CSV para tabelas

#### Método 1: Explorar os dados primeiro
```sql
SELECT *
FROM csv.`/Volumes/smart_claims_dev/00_landing/sql_server/claims.csv`
```

**Explicação:**
- `csv.` = "Leia como arquivo CSV"
- Backticks `` ` `` = Indicam um caminho de arquivo
- Útil para **ver os dados antes** de criar a tabela

#### Método 2: CREATE TABLE AS SELECT (CTAS)
```sql
CREATE TABLE smart_claims_dev.01_bronze.claims
AS SELECT *
FROM read_files(
  '/Volumes/smart_claims_dev/00_landing/sql_server/claims.csv',
  format => 'csv'
)
```

**Explicação passo a passo:**
1. `CREATE TABLE` = "Crie uma tabela"
2. `AS SELECT` = "Usando os dados que vou buscar"
3. `read_files()` = Função que lê arquivos
4. `format => 'csv'` = "O arquivo é CSV"

**Quando usar:** Carga inicial de dados (primeira vez)

#### Método 3: Python com PySpark
```python
# 1. Ler o arquivo CSV
df = (
    spark.read
    .format("csv")
    .option("header", True)  # Primeira linha é cabeçalho
    .load("/Volumes/smart_claims_dev/00_landing/sql_server/claims.csv")
)

# 2. Salvar como tabela
df.write.mode("overwrite").saveAsTable("smart_claims_dev.01_bronze.claims")

# 3. Ler a tabela criada
claims_table = spark.table("smart_claims_dev.01_bronze.claims")
display(claims_table)
```

**Explicação:**
- `spark.read` = Objeto para ler dados
- `.option("header", True)` = "A primeira linha tem os nomes das colunas"
- `.mode("overwrite")` = "Se a tabela existir, substitua"
- `display()` = Mostra os dados na tela

**Quando usar:** Quando precisa de mais controle ou transformações em Python

#### Método 4: COPY INTO (Incremental)
```sql
-- Primeiro: Criar a estrutura da tabela
CREATE TABLE smart_claims_dev.01_bronze.claims (
  claim_no STRING,
  policy_no STRING,
  claim_date STRING,
  -- ... outras colunas
)

-- Depois: Copiar os dados
COPY INTO smart_claims_dev.01_bronze.claims
FROM '/Volumes/smart_claims_dev/00_landing/sql_server/claims.csv'
FILEFORMAT = CSV
FORMAT_OPTIONS ('header'='true')
```

**Explicação:**
- `COPY INTO` = "Copie dados para dentro da tabela"
- **Vantagem:** Só copia arquivos novos (não duplica)
- **Quando usar:** Cargas diárias/incrementais

---

### Notebook 4: Auto Loader + Lakeflow (Streaming)

**O que faz:** Ingestão automática e contínua de dados

#### Conceito: Streaming vs Batch

**Batch (Notebooks anteriores):**
- Você executa manualmente
- Processa tudo de uma vez
- Como baixar um filme completo

**Streaming (Auto Loader):**
- Executa automaticamente
- Processa novos arquivos conforme chegam
- Como assistir Netflix (streaming contínuo)

#### Comando: Criar Streaming Table
```sql
CREATE OR REFRESH STREAMING LIVE TABLE smart_claims_dev.01_bronze.claims
AS
SELECT
  *,
  _metadata.file_path AS input_file,
  _metadata.file_modification_time AS input_file_mtime,
  current_timestamp() AS ingested_at
FROM cloud_files(
  "/Volumes/smart_claims_dev/00_landing/sql_server",
  "csv",
  map(
    "header", "true",
    "cloudFiles.inferColumnTypes", "true",
    "cloudFiles.schemaEvolutionMode", "addNewColumns",
    "pathGlobFilter", "claims.csv"
  )
)
```

**Explicação detalhada:**

1. **CREATE OR REFRESH STREAMING LIVE TABLE**
   - `STREAMING` = Processa dados continuamente
   - `LIVE` = Sempre atualizada
   - `OR REFRESH` = Atualiza se já existir

2. **Colunas extras de metadados:**
   ```sql
   _metadata.file_path AS input_file
   ```
   - Guarda de qual arquivo veio cada linha
   - Útil para rastreabilidade

   ```sql
   current_timestamp() AS ingested_at
   ```
   - Registra quando o dado foi ingerido
   - Útil para auditoria

3. **cloud_files() - O Auto Loader:**
   - `"/Volumes/.../sql_server"` = Pasta para monitorar
   - `"csv"` = Tipo de arquivo
   
4. **Configurações (map):**
   ```sql
   "header", "true"
   ```
   - Primeira linha é cabeçalho
   
   ```sql
   "cloudFiles.inferColumnTypes", "true"
   ```
   - Detecta automaticamente se é número, texto, data, etc.
   
   ```sql
   "cloudFiles.schemaEvolutionMode", "addNewColumns"
   ```
   - Se aparecer coluna nova no CSV, adiciona automaticamente
   
   ```sql
   "pathGlobFilter", "claims.csv"
   ```
   - Só processa arquivos com nome "claims.csv"

#### Agendamento
```sql
CREATE OR REFRESH STREAMING TABLE smart_claims_dev.01_bronze.claims
SCHEDULE EVERY 1 WEEK
```

**Explicação:**
- `SCHEDULE EVERY 1 WEEK` = Executa automaticamente toda semana
- Pode usar: `1 HOUR`, `1 DAY`, `1 WEEK`

---

## 🔄 Fluxo Completo do Projeto

```
1. PREPARAÇÃO
   ├── Criar Catálogo (smart_claims_dev)
   ├── Criar Schemas (00_landing, 01_bronze, 02_silver, 03_gold)
   └── Criar Volume (sql_server)

2. UPLOAD DE DADOS
   └── Colocar claims.csv no volume

3. INGESTÃO (Escolha uma opção)
   ├── Opção A: CTAS (carga única)
   ├── Opção B: Python (mais controle)
   ├── Opção C: COPY INTO (incremental manual)
   └── Opção D: Auto Loader (streaming automático) ⭐ Recomendado

4. RESULTADO
   └── Tabela smart_claims_dev.01_bronze.claims pronta para uso
```

---

## 📊 Dados do Projeto

O arquivo `claims.csv` contém informações de sinistros de seguros:

| Coluna | Descrição | Exemplo |
|--------|-----------|---------|
| claim_no | Número do sinistro | CLM001 |
| policy_no | Número da apólice | POL12345 |
| claim_date | Data do sinistro | 2024-01-15 |
| injury | Valor de danos pessoais | 5000.00 |
| property | Valor de danos materiais | 15000.00 |
| vehicle | Valor de danos ao veículo | 8000.00 |
| total | Valor total | 28000.00 |
| collision_type | Tipo de colisão | Rear-end |
| suspicious_activity | Atividade suspeita? | Yes/No |

---

## 🎓 Comparação dos Métodos

| Método | Quando Usar | Vantagens | Desvantagens |
|--------|-------------|-----------|--------------|
| **CTAS** | Carga inicial única | Simples e rápido | Não detecta novos arquivos |
| **Python** | Transformações complexas | Muito flexível | Mais código |
| **COPY INTO** | Cargas diárias manuais | Evita duplicatas | Precisa executar manualmente |
| **Auto Loader** | Produção (automático) | Totalmente automático, schema evolution | Mais complexo de configurar |

---

## 🚀 Como Executar

### Passo 1: Preparar o ambiente
```sql
-- Execute o Notebook 1
-- Cria: Catálogo + Schemas
```

### Passo 2: Criar volumes
```sql
-- Execute o Notebook 2
-- Cria: Volume para arquivos
```

### Passo 3: Upload do arquivo
```python
# Via interface do Databricks ou comando:
dbutils.fs.cp("file:/local/claims.csv", 
              "/Volumes/smart_claims_dev/00_landing/sql_server/claims.csv")
```

### Passo 4: Escolher método de ingestão
```sql
-- Para começar: Use Notebook 3 (CTAS)
-- Para produção: Use Notebook 4 (Auto Loader)
```

---

## 💡 Dicas e Boas Práticas

### 1. Sempre use IF NOT EXISTS
```sql
CREATE CATALOG IF NOT EXISTS smart_claims_dev
```
✅ Evita erros se já existir

### 2. Documente com COMMENT
```sql
CREATE SCHEMA smart_claims_dev.01_bronze
COMMENT 'Camada Bronze - dados brutos'
```
✅ Facilita entender depois

### 3. Adicione metadados de auditoria
```sql
SELECT 
  *,
  current_timestamp() AS ingested_at,
  _metadata.file_path AS source_file
```
✅ Rastreabilidade completa

### 4. Use nomenclatura padronizada
```
00_landing  ← Números ajudam na ordenação
01_bronze
02_silver
03_gold
```

### 5. Prefira Auto Loader para produção
- Detecta novos arquivos automaticamente
- Schema evolution
- Checkpoint para não reprocessar

---

## 🔍 Comandos Úteis para Exploração

```sql
-- Ver catálogos disponíveis
SHOW CATALOGS;

-- Ver schemas de um catálogo
SHOW SCHEMAS IN smart_claims_dev;

-- Ver tabelas de um schema
SHOW TABLES IN smart_claims_dev.01_bronze;

-- Ver volumes
SHOW VOLUMES IN smart_claims_dev.00_landing;

-- Listar arquivos em um volume
LIST '/Volumes/smart_claims_dev/00_landing/sql_server';

-- Ver estrutura de uma tabela
DESCRIBE TABLE smart_claims_dev.01_bronze.claims;

-- Ver detalhes completos
DESCRIBE TABLE EXTENDED smart_claims_dev.01_bronze.claims;

-- Ver dados
SELECT * FROM smart_claims_dev.01_bronze.claims LIMIT 10;

-- Contar registros
SELECT COUNT(*) FROM smart_claims_dev.01_bronze.claims;
```

---

## 🎯 Próximos Passos

Depois de dominar a ingestão (Bronze), você pode:

1. **Criar camada Silver:**
   - Limpar dados (remover nulos, duplicatas)
   - Converter tipos (string → date, decimal)
   - Validar regras de negócio

2. **Criar camada Gold:**
   - Agregações (total por mês, por tipo)
   - Joins com outras tabelas
   - Métricas de negócio

3. **Adicionar governança:**
   - Controle de acesso (quem pode ver o quê)
   - Data quality checks
   - Lineage (rastreamento de origem)

---

## 📚 Glossário

- **Catalog**: Nível mais alto de organização no Unity Catalog
- **Schema**: Agrupamento lógico de tabelas (equivalente a "database")
- **Volume**: Espaço para armazenar arquivos não estruturados
- **Delta Lake**: Formato de armazenamento transacional (ACID)
- **Auto Loader**: Ferramenta para ingestão incremental automática
- **Streaming Table**: Tabela que processa dados continuamente
- **CTAS**: CREATE TABLE AS SELECT - cria tabela a partir de query
- **Schema Evolution**: Adaptação automática quando estrutura muda

---

## 👤 Autor

**Vanessa Prado**
- GitHub: [@euvanessa-prado](https://github.com/euvanessa-prado)

---

## 📄 Licença

Projeto educacional - livre para uso e aprendizado! 🎓
