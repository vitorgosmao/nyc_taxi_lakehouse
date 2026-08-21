# NYC Taxi Lakehouse

Pipeline de dados em arquitetura medallion (Bronze → Silver → Gold) construído com PySpark e Delta Lake no Databricks, utilizando o dataset público de corridas de táxi da cidade de Nova York (NYC TLC Trip Record Data).

## Objetivo

Processar um ano completo de dados de corridas de táxi de forma incremental, simulando um pipeline de ingestão mensal, e responder a uma pergunta analítica de negócio: **em quais combinações de zona e horário existe maior desalinhamento entre oferta e demanda de corridas**, usando volume de corridas, tempo médio de espera implícito e comportamento de tarifa/gorjeta como proxies.

## Stack

- **Processamento**: PySpark (DataFrame API + Spark SQL)
- **Armazenamento**: Delta Lake, gerenciado via Unity Catalog (catálogos, schemas e volumes)
- **Plataforma**: Databricks Free Edition (compute serverless)
- **Versionamento**: Git folder do Databricks, sincronizado com este repositório
- **Fonte de dados**: [NYC TLC Trip Record Data](https://www.nyc.gov/site/tlc/about/tlc-trip-record-data.page) — arquivos parquet mensais, ano completo

## Arquitetura

O projeto segue o padrão medallion, com cada camada representada por um schema dedicado no Unity Catalog:

```
nyc_taxi (catálogo)
├── bronze
│   ├── raw_files          → volume com os parquet mensais brutos, como baixados da fonte
│   ├── trips_bronze        → tabela Delta, carga incremental via MERGE INTO
│   └── _ingestion_log      → tabela de controle, registra quais meses já foram processados
├── silver
│   └── trips_silver        → dados limpos, tipados e deduplicados, com regras de qualidade aplicadas
└── gold
    └── demand_supply_gold  → agregações analíticas por zona e horário, prontas para consumo
```

### Bronze

Ingestão incremental simulada: cada execução processa um mês específico do ano, via parâmetro de widget. A carga usa `MERGE INTO` para garantir idempotência — reprocessar o mesmo mês não gera duplicidade. A tabela `_ingestion_log` registra o histórico de cargas, funcionando como um checkpoint manual na ausência de um orquestrador.

### Silver

Aplica limpeza, tipagem correta de colunas, remoção de duplicatas e validações de qualidade (ex: descarte de corridas com distância ou duração inconsistente, coordenadas ou zonas inválidas).

### Gold

Camada analítica final, construída com Spark SQL avançado (window functions, CTEs), agregando os dados por zona e faixa horária para sustentar a análise de desalinhamento entre oferta e demanda.

## Estrutura do repositório

```
nyc_taxi_lakehouse/
├── README.md
├── notebooks/
│   ├── 00_setup_and_config
│   ├── 01_bronze_ingestion
│   ├── 02_silver_transformation
│   ├── 03_gold_demand_supply
│   └── 04_ingestion_control
├── sql/
│   └── gold_queries.sql
└── docs/
    └── data_dictionary.md
```

## Como executar

1. Clonar este repositório como Git folder no Databricks Free Edition.
2. Garantir que o catálogo `nyc_taxi` e os schemas `bronze`, `silver` e `gold` existem no Unity Catalog, junto com o volume `nyc_taxi.bronze.raw_files`.
3. Executar `00_setup_and_config` para validar a estrutura e definir os widgets de parâmetro (mês/ano).
4. Executar `01_bronze_ingestion` para cada mês do ano, sequencialmente, para simular a carga incremental.
5. Executar `02_silver_transformation` e `03_gold_demand_supply` para completar o pipeline.
6. Consultar `sql/gold_queries.sql` para as análises finais sobre desalinhamento de oferta e demanda.

## Status

Projeto em desenvolvimento.
