# 🌦️ ATMOS — Pipeline de Dados Climáticos no Databricks

## Resumo

O **ATMOS** é um pipeline de dados climáticos de nível produtivo construído sobre **Databricks + Azure Data Lake Storage Gen2**. Ele ingere dados meteorológicos de duas fontes — o **INMET** (Instituto Nacional de Meteorologia) e a **Visual Crossing** — e os processa através da **Arquitetura Medallion** (Bronze → Silver → Gold), culminando em um dashboard analítico de perfil sazonal. O projeto foi desenhado com foco em idempotência, qualidade de dados e um esquema canônico unificado que permite comparar fontes heterogêneas na mesma estrutura.

---

## 📋 Sumário

- [Visão Geral da Arquitetura](#-visão-geral-da-arquitetura)
- [Estrutura de Diretórios](#-estrutura-de-diretórios)
- [Descrição das Camadas](#-descrição-das-camadas)
  - [Bronze — Ingestão Bruta](#bronze--ingestão-bruta)
  - [Silver — INMET](#silver--inmet)
  - [Silver — Visual Crossing](#silver--visual-crossing)
  - [Silver — Unificado](#silver--unificado)
  - [Gold — Perfil Sazonal Mensal](#gold--perfil-sazonal-mensal)
  - [Dashboard](#dashboard)
- [Esquema Canônico Climático](#-esquema-canônico-climático)
- [Stack Tecnológica](#-stack-tecnológica)
- [Fluxo do Pipeline (Databricks Workflow)](#-fluxo-do-pipeline-databricks-workflow)
- [Como Executar e Fazer Deploy](#-como-executar-e-fazer-deploy)
- [Decisões de Design](#-decisões-de-design)
- [Pré-requisitos](#-pré-requisitos)

---

## 🏗️ Visão Geral da Arquitetura

O ATMOS segue a **Arquitetura Medallion**, um padrão de organização de data lakehouse que separa os dados em três camadas de qualidade crescente. Cada camada tem responsabilidades bem definidas e nenhuma transformação "pula" etapas.

```
╔══════════════════════════════════════════════════════════════════╗
║                        FONTES DE DADOS                           ║
╠══════════════════╦═══════════════════════════════════════════════╣
║  INMET (CSV)     ║          Visual Crossing (JSON)               ║
║  Estações + dados║          Histórico / Previsão                  ║
║  horários        ║          Dados diários por cidade             ║
╚══════════════════╩═══════════════════════════════════════════════╝
          │                              │
          └──────────────┬───────────────┘
                         ▼
╔══════════════════════════════════════════════════════════════════╗
║                    CAMADA BRONZE                                 ║
║              Auto Loader (cloudFiles)                            ║
║  ┌─────────────────────────────────────────────────────────┐     ║
║  │  atmos.bronze.inmet_estacao_raw                         │     ║
║  │  atmos.bronze.inmet_microdados_raw                      │     ║
║  │  atmos.bronze.visual_crossing_raw                       │     ║
║  └─────────────────────────────────────────────────────────┘     ║
║       Delta Lake · Particionado por ingestion_date               ║
╚══════════════════════════════════════════════════════════════════╝
                         │
                         ▼
╔══════════════════════════════════════════════════════════════════╗
║                    CAMADA SILVER                                  ║
║          Normalização · Validação · Esquema Canônico             ║
║  ┌─────────────────────────────────────────────────────────┐     ║
║  │  atmos.silver.climate_inmet                             │     ║
║  │  atmos.silver.climate_visual_crossing                   │     ║
║  │  atmos.silver.climate_unified  ◄── unionByName()        │     ║
║  └─────────────────────────────────────────────────────────┘     ║
╚══════════════════════════════════════════════════════════════════╝
                         │
                         ▼
╔══════════════════════════════════════════════════════════════════╗
║                     CAMADA GOLD                                   ║
║              Agregação em dois estágios (CTEs)                   ║
║  ┌─────────────────────────────────────────────────────────┐     ║
║  │  atmos.gold.perfil_sazonal_mensal                       │     ║
║  │  Diário (AVG/MAX/MIN/SUM) → Mensal (perfil climático)   │     ║
║  └─────────────────────────────────────────────────────────┘     ║
╚══════════════════════════════════════════════════════════════════╝
                         │
                         ▼
╔══════════════════════════════════════════════════════════════════╗
║                      DASHBOARD                                    ║
║        Databricks SQL · perfil_sazonal_mensal                    ║
║        Precipitação por ano/cidade · Última atualização          ║
╚══════════════════════════════════════════════════════════════════╝
```

---

## 📁 Estrutura de Diretórios

```
atmos-databricks/
├── 01 - Create Bronze/
│   └── ingest_bronze.ipynb           # Notebook parametrizado de ingestão via Auto Loader
├── 02 - Create Silver/
│   ├── transform_inmet.ipynb         # Normalização e validação dos dados INMET
│   ├── transform_visual_crossing.ipynb  # Parse do JSON aninhado da Visual Crossing
│   └── transform_unified.ipynb       # União das duas fontes Silver
├── 03 - Create Gold/
│   └── perfil_sazonal_mensal.ipynb   # Agregação diária → mensal com CTEs
├── 04 - Dashboard/
│   └── Dashboard ATMOS.lvdash.json   # Definição do dashboard Databricks SQL
├── 05 - Pipeline/
│   └── Pipeline_ATMOS.yml            # Definição do Databricks Workflow (YAML)
└── README.md
```

Cada diretório numerado corresponde a uma fase do pipeline. A numeração não é apenas organizacional — ela reflete a ordem de dependência das camadas.

---

## 🔄 Descrição das Camadas

### Bronze — Ingestão Bruta

**Notebook:** `01 - Create Bronze/ingest_bronze.ipynb`

A camada Bronze é o ponto de entrada de todos os dados. O princípio central aqui é a **fidelidade**: os dados são armazenados o mais próximo possível do formato original, com a adição de metadados de rastreabilidade.

**Mecanismo de ingestão:** o notebook utiliza o **Auto Loader** (`cloudFiles`) do Databricks para leitura incremental e contínua de arquivos no Azure Data Lake. O Auto Loader mantém checkpoints que garantem que cada arquivo seja processado exatamente uma vez, mesmo em re-execuções.

**Parâmetros do notebook (recebidos via Databricks Workflow):**

| Parâmetro | Descrição |
|---|---|
| `source_system` | Identificador da fonte (`inmet`, `visual_crossing`) |
| `source_path` | Caminho `abfss://` dos arquivos de entrada |
| `file_format` | Formato do arquivo (`csv`, `json`, `parquet`) |
| `target_table` | Tabela Delta de destino |
| `ingestion_date` | Data da ingestão (usada como partição) |
| `schema_location` | Caminho para armazenar o schema inferido |
| `checkpoint_location` | Caminho do checkpoint do Auto Loader |

**Metadados adicionados a cada registro:**

- `ingestion_timestamp` — momento exato da ingestão
- `source_system` — sistema de origem
- `file_name` — nome do arquivo de origem (rastreabilidade)
- `ingestion_date` — data de ingestão (chave de partição)

**Localização de armazenamento:** `abfss://` → `/Volumes/atmos/bronze/landing/`

**Modo de escrita:** Append via Spark Structured Streaming com `trigger(availableNow=True)`, que processa todos os arquivos pendentes e para — comportamento equivalente a um batch incremental.

**Tabelas produzidas:**

- `atmos.bronze.inmet_estacao_raw` — metadados das estações meteorológicas
- `atmos.bronze.inmet_microdados_raw` — medições horárias das estações
- `atmos.bronze.visual_crossing_raw` — dados climáticos da Visual Crossing

---

### Silver — INMET

**Notebook:** `02 - Create Silver/transform_inmet.ipynb`

Esta etapa transforma os dados brutos do INMET no **esquema canônico climático** do projeto (detalhado na seção [Esquema Canônico Climático](#-esquema-canônico-climático)).

**Operações realizadas:**

1. **Normalização de colunas** — renomeia e reorienta os campos do CSV do INMET para o esquema canônico, aplicando conversões de tipo quando necessário.
2. **Enriquecimento com metadados de estação** — realiza um join com `inmet_estacao_raw` para incluir o nome da cidade associada ao código da estação.
3. **Validação de qualidade de dados** — filtra registros com valores fora de intervalos meteorologicamente plausíveis (temperatura, umidade, pressão, velocidade do vento, etc.). Registros inválidos são descartados ou sinalizados antes da gravação.
4. **Deduplicação** — remove registros duplicados com base na chave composta `(data_observacao, hora_observacao, id_estacao)`.

**Tabela produzida:** `atmos.silver.climate_inmet`

---

### Silver — Visual Crossing

**Notebook:** `02 - Create Silver/transform_visual_crossing.ipynb`

A Visual Crossing entrega dados em formato JSON com estrutura aninhada. Esta etapa resolve o aninhamento e mapeia os campos para o esquema canônico.

**Operações realizadas:**

1. **Parse do JSON aninhado** — o array `days` dentro do JSON é extraído com schema explícito usando `from_json` + `explode`, expandindo cada dia como uma linha individual.
2. **Extração do nome da cidade** — o nome da cidade é derivado do nome do arquivo via expressão regular com o padrão `atmos_[cidade]_*`. Isso elimina a necessidade de um campo de cidade no payload JSON.
3. **Mapeamento para o esquema canônico** — os campos da Visual Crossing são mapeados para as colunas equivalentes. Como a Visual Crossing fornece dados **diários** (e não horários), a coluna `hora_observacao` é definida como `null`.

**Tabela produzida:** `atmos.silver.climate_visual_crossing`

---

### Silver — Unificado

**Notebook:** `02 - Create Silver/transform_unified.ipynb`

Esta etapa é a mais simples em termos de código, mas a mais importante em termos de valor analítico: ela une as duas tabelas Silver em uma única visão consolidada.

**Operação:** `unionByName()` entre `climate_inmet` e `climate_visual_crossing`. O uso de `unionByName` (em vez de `union`) garante que os dados sejam alinhados por nome de coluna, não por posição, tornando o código robusto a futuras adições de campos.

Como ambas as tabelas já seguem o esquema canônico, nenhuma transformação adicional é necessária. Campos não disponíveis em uma fonte ficam como `null`.

**Tabela produzida:** `atmos.silver.climate_unified`

---

### Gold — Perfil Sazonal Mensal

**Notebook:** `03 - Create Gold/perfil_sazonal_mensal.ipynb`

A camada Gold produz a **tabela analítica final**: o perfil climático mensal por cidade e estação. A agregação é feita em **dois estágios** usando CTEs (Common Table Expressions), uma decisão de design detalhada na seção [Decisões de Design](#-decisões-de-design).

**Estágio 1 — CTE `diario`**

Agrega os dados horários do INMET para granularidade diária:

- Temperatura: `AVG`, `MAX`, `MIN`
- Precipitação: `SUM`
- Umidade: `AVG`
- Pressão: `AVG`

**Estágio 2 — CTE `mensal`**

Agrega os resultados diários para o perfil mensal, combinando os dados do INMET (via CTE diário) com os dados diários da Visual Crossing.

**Colunas produzidas na tabela Gold:**

| Coluna | Descrição |
|---|---|
| `temperatura_media_c` | Temperatura média mensal (°C) |
| `temperatura_maxima_media_c` | Média das temperaturas máximas diárias (°C) |
| `temperatura_minima_media_c` | Média das temperaturas mínimas diárias (°C) |
| `temperatura_maxima_abs_c` | Temperatura máxima absoluta do mês (°C) |
| `temperatura_minima_abs_c` | Temperatura mínima absoluta do mês (°C) |
| `precipitacao_total_mm` | Precipitação total acumulada no mês (mm) |
| `umidade_media_pct` | Umidade relativa média mensal (%) |
| `pressao_media_hpa` | Pressão atmosférica média mensal (hPa) |
| `amplitude_termica_media_c` | Amplitude térmica média diária (°C) |
| `dias_com_dados` | Número de dias com registros válidos no mês |
| `timestamp_processamento` | Momento em que o registro foi gerado |

**Tabela produzida:** `atmos.gold.perfil_sazonal_mensal`

---

### Dashboard

**Arquivo:** `04 - Dashboard/Dashboard ATMOS.lvdash.json`

Dashboard nativo do **Databricks SQL** que consulta diretamente `atmos.gold.perfil_sazonal_mensal`. Exibe:

- Timestamp da última atualização dos dados
- Precipitação total por ano e cidade
- Perfis de temperatura sazonal

O arquivo `.lvdash.json` contém a definição completa do dashboard e pode ser importado pela UI do Databricks ou via CLI.

---

## 📊 Esquema Canônico Climático

O esquema canônico é o contrato central do ATMOS. Todo dado que entra na camada Silver, independentemente da fonte, deve ser convertido para este formato. Isso garante que `climate_inmet` e `climate_visual_crossing` sejam uníveis sem transformações adicionais.

| Coluna | Tipo | Descrição |
|---|---|---|
| `data_observacao` | `Date` | Data da observação meteorológica |
| `hora_observacao` | `String` | Hora da observação (`null` para dados diários) |
| `id_estacao` | `String` | Código da estação meteorológica |
| `cidade` | `String` | Nome da cidade associada |
| `sistema_origem` | `String` | Fonte do dado (`"inmet"` ou `"visual_crossing"`) |
| `temperatura_c` | `Double` | Temperatura média ou corrente (°C) |
| `temperatura_max_c` | `Double` | Temperatura máxima (°C) |
| `temperatura_min_c` | `Double` | Temperatura mínima (°C) |
| `ponto_orvalho_c` | `Double` | Ponto de orvalho (°C) |
| `umidade_pct` | `Double` | Umidade relativa do ar (%) |
| `umidade_max_pct` | `Double` | Umidade relativa máxima (%) |
| `umidade_min_pct` | `Double` | Umidade relativa mínima (%) |
| `pressao_hpa` | `Double` | Pressão atmosférica (hPa) |
| `precipitacao_mm` | `Double` | Precipitação (mm) |
| `radiacao_solar_wm2` | `Double` | Radiação solar global (W/m²) |
| `velocidade_vento_ms` | `Double` | Velocidade do vento (m/s) |
| `rajada_vento_ms` | `Double` | Velocidade de rajada de vento (m/s) |
| `direcao_vento_graus` | `Double` | Direção do vento (graus) |
| `indice_uv` | `Double` | Índice UV |
| `data_ingestao` | `Date` | Data de ingestão (chave de partição) |

> **Nota sobre campos nulos:** colunas não disponíveis em uma determinada fonte são armazenadas como `null`, não omitidas. Isso preserva a consistência do schema e permite que filtros e agregações funcionem uniformemente sobre `climate_unified`.

---

## 🛠️ Stack Tecnológica

| Componente | Tecnologia |
|---|---|
| Plataforma de processamento | Databricks Lakehouse |
| Armazenamento | Azure Data Lake Storage Gen2 (`abfss://`) |
| Formato de arquivo | Delta Lake (transações ACID, time travel) |
| Ingestão incremental | Auto Loader (`cloudFiles`) com evolução de schema |
| Processamento | PySpark + Spark SQL |
| Orquestração | Databricks Workflows (definição em YAML) |
| Visualização | Databricks SQL Dashboard |

---

## ⚙️ Fluxo do Pipeline (Databricks Workflow)

**Arquivo de definição:** `05 - Pipeline/Pipeline_ATMOS.yml`

O pipeline é orquestrado como um **Databricks Workflow** com 8 tasks, cada uma com dependências explícitas. O grafo de dependências garante que cada tarefa só execute após seus pré-requisitos terem sido concluídos com sucesso.

```
bronze_inmet_estacoes ──┐
                        ├──► silver_inmet ──────────────┐
bronze_inmet_microdados─┘                               │
                                                        ├──► silver_unified ──► gold_perfil_sazonal_mensal ──► Dashboard_ATMOS
bronze_visualcrossing ──────► silver_visualcrossing ────┘
```

**Descrição das tasks:**

| # | Task | Dependências | Notebook |
|---|---|---|---|
| 1 | `bronze_inmet_estacoes` | — | `ingest_bronze.ipynb` |
| 2 | `bronze_inmet_microdados` | — | `ingest_bronze.ipynb` |
| 3 | `bronze_visualcrossing` | — | `ingest_bronze.ipynb` |
| 4 | `silver_inmet` | tasks 1 e 2 | `transform_inmet.ipynb` |
| 5 | `silver_visualcrossing` | task 3 | `transform_visual_crossing.ipynb` |
| 6 | `silver_unified` | tasks 4 e 5 | `transform_unified.ipynb` |
| 7 | `gold_perfil_sazonal_mensal` | task 6 | `perfil_sazonal_mensal.ipynb` |
| 8 | `Dashboard_ATMOS` | task 7 | *(atualização do dashboard)* |

> As tasks 1, 2 e 3 não têm dependências entre si e podem — e devem — ser executadas em paralelo pelo Workflow para reduzir o tempo total de execução.

---

## 🚀 Como Executar e Fazer Deploy

### Pré-requisito de configuração

Antes de executar o pipeline, certifique-se de que os seguintes recursos estão provisionados no Azure e no Databricks:

- Workspace Databricks com Unity Catalog habilitado
- Storage Account com ADLS Gen2 e container `atmos` criado
- Credenciais de acesso ao ADLS configuradas como Databricks Secret ou Storage Credential no Unity Catalog
- Catálogo `atmos` criado no Unity Catalog com os schemas `bronze`, `silver` e `gold`
- Volume `atmos.bronze.landing` criado para receber os arquivos de entrada

### 1. Upload dos dados brutos

Faça o upload dos arquivos de entrada para o volume de landing no ADLS:

```
/Volumes/atmos/bronze/landing/inmet/estacoes/      ← CSVs de estações
/Volumes/atmos/bronze/landing/inmet/microdados/    ← CSVs de microdados horários
/Volumes/atmos/bronze/landing/visual_crossing/     ← JSONs nomeados como atmos_[cidade]_*.json
```

### 2. Deploy do Workflow via CLI

Com a [Databricks CLI](https://docs.databricks.com/dev-tools/cli/index.html) configurada:

```bash
# Autenticar na CLI (se ainda não estiver autenticado)
databricks configure --token

# Fazer o deploy do pipeline
databricks bundle deploy --target production

# Ou executar diretamente o workflow
databricks workflows run Pipeline_ATMOS
```

### 3. Execução manual pela UI

1. Acesse o workspace Databricks
2. Navegue até **Workflows** no menu lateral
3. Localize `Pipeline_ATMOS` e clique em **Run now**
4. Acompanhe o progresso de cada task no grafo de execução

### 4. Execução individual de notebooks (desenvolvimento)

Cada notebook pode ser executado individualmente no Databricks com parâmetros manuais. O notebook de Bronze, por exemplo, aceita os seguintes widgets:

```python
dbutils.widgets.text("source_system", "inmet")
dbutils.widgets.text("source_path", "abfss://atmos@<storage>.dfs.core.windows.net/landing/inmet/microdados/")
dbutils.widgets.text("file_format", "csv")
dbutils.widgets.text("target_table", "atmos.bronze.inmet_microdados_raw")
dbutils.widgets.text("ingestion_date", "2024-01-15")
```

### 5. Importar o Dashboard

1. No workspace Databricks, acesse **SQL** → **Dashboards**
2. Clique em **Import** e selecione o arquivo `04 - Dashboard/Dashboard ATMOS.lvdash.json`
3. Configure o SQL Warehouse a ser utilizado nas queries do dashboard

---

## 🧠 Decisões de Design

### Idempotência

Cada execução do pipeline pode ser repetida com segurança sem gerar duplicatas ou dados inconsistentes. Isso é garantido por três mecanismos complementares:

- **Auto Loader com checkpoint:** o Auto Loader rastreia quais arquivos já foram processados. Re-executar a ingestão Bronze não reprocessa arquivos já consumidos.
- **Deduplicação na Silver:** a etapa INMET remove explicitamente duplicatas com base na chave natural `(data_observacao, hora_observacao, id_estacao)`.
- **Substituição de partição:** a escrita nas tabelas Silver e Gold usa substituição de partição por `data_ingestao`, garantindo que apenas a partição da data corrente seja reescrita.

### Evolução de Schema

O Auto Loader é configurado com `cloudFiles.schemaEvolutionMode = "addNewColumns"`. Quando o INMET ou a Visual Crossing adicionam novos campos nos arquivos, eles são automaticamente incorporados ao schema da tabela Bronze sem necessidade de intervenção manual. Isso evita falhas de pipeline por incompatibilidade de schema.

### Qualidade de Dados

Antes de escrever na Silver, todos os campos meteorológicos são validados contra intervalos plausíveis (ex: temperatura entre -40°C e 60°C, umidade entre 0% e 100%). Registros que violam essas regras são descartados, prevenindo que outliers corrompidos contaminem a Gold e o dashboard.

### Esquema Canônico

A criação de um esquema canônico unificado é a principal decisão arquitetural do projeto. O INMET e a Visual Crossing têm nomenclaturas, granularidades (horária vs. diária) e estruturas de arquivo completamente diferentes. Ao forçar ambos para o mesmo contrato na Silver, a camada Gold e o dashboard ficam completamente agnósticos à fonte, simplificando toda análise futura.

Campos não disponíveis em uma fonte são armazenados como `null`, e não omitidos. Isso mantém o schema estável e previsível para todas as queries downstream.

### Agregação Gold em Dois Estágios

O INMET fornece dados **horários** e a Visual Crossing fornece dados **diários**. Para calcular um perfil mensal consistente entre as duas fontes, os dados do INMET precisam primeiro ser agregados para granularidade diária — caso contrário, uma soma de precipitação mensal do INMET somaria 24x mais valores do que a Visual Crossing para o mesmo período.

A solução adotada é uma CTE de dois estágios:

```
INMET horário → CTE diario (AVG/MAX/MIN/SUM por dia) → CTE mensal
Visual Crossing diário ──────────────────────────────→ CTE mensal
```

Isso garante paridade matemática entre as fontes ao calcular médias e acumulados mensais.

---

## ✅ Pré-requisitos

Para trabalhar neste projeto, o engenheiro de dados deve ter familiaridade com:

- **Databricks Lakehouse Platform** — workspaces, notebooks, clusters, Unity Catalog
- **Apache Spark (PySpark)** — DataFrames, Spark SQL, Structured Streaming
- **Delta Lake** — formato de arquivo, transações ACID, operações de merge/overwrite
- **Auto Loader** — mecanismo de ingestão incremental do Databricks
- **Azure Data Lake Storage Gen2** — estrutura de containers, protocolo `abfss://`
- **Databricks Workflows** — definição e orquestração de pipelines
- **Arquitetura Medallion** — padrão Bronze/Silver/Gold

---

## 📚 Referências

- [Databricks Auto Loader — Documentação Oficial](https://docs.databricks.com/ingestion/auto-loader/index.html)
- [Delta Lake — Documentação Oficial](https://docs.delta.io/latest/index.html)
- [Databricks Workflows — Documentação Oficial](https://docs.databricks.com/workflows/index.html)
- [Unity Catalog — Documentação Oficial](https://docs.databricks.com/data-governance/unity-catalog/index.html)
- [INMET — Banco de Dados Meteorológicos](https://bdmep.inmet.gov.br/)
- [Visual Crossing Weather API](https://www.visualcrossing.com/weather-api)
- [Medallion Architecture — Databricks](https://www.databricks.com/glossary/medallion-architecture)

---

*Última atualização: Abril de 2026*
