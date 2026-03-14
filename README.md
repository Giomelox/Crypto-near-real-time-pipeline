# Crypto Near Real Time Pipeline

## Este projeto implementa um pipeline de dados orientado a eventos para ingestão e processamento de dados de mercado de criptomoedas, simulando um cenário near real-time em ambiente AWS. Em um ambiente produtivo, a ingestão contínua seria realizada com Amazon Kinesis e processamento streaming dedicado. Neste projeto, o comportamento foi simulado utilizando agendamentos periódicos e eventos do S3, mantendo os princípios arquiteturais de um pipeline real-time.

# Fluxo Geral

EventBridge (1 min schedule) → Lambda (API ingestion) → S3 Bronze (raw JSON) → EventBridge (via S3 Event) → Glue Workflow → Glue Job Silver (typed + partitioned parquet) → Glue Job Gold (hourly & daily analytics) → AWS Glue Data Catalog + AWS Athena

# Decisões Arquiteturais

- A ingestão foi implementada via polling (EventBridge 1 min) para simular streaming em ambiente gratuito.

- O processamento Silver é orientado a evento (S3 → EventBridge → Glue Workflow).

- A camada Gold é processada em batch controlado (Hourly/Daily) para simular agregações near real-time.

- O uso de Parquet foi escolhido por otimização de leitura analítica e compressão columnar.

- O particionamento é baseado no timestamp da fonte (e não da ingestão) para manter consistência temporal.

# Serviços Utilizados:

- AWS IAM Role + IAM Policy
- AWS S3
- AWS Lambda
- AWS EventBridge
- AWS Glue Job
- AWS Glue Workflow
- AWS Glue Data Catalog
- AWS Athena
- AWS CloudWatch

# O projeto segue o padrão:

- Bronze → dados brutos

- Silver → dados tratados e tipados

- Gold → dados agregados e prontos para analytics

# Esse modelo garante:

- Governança

- Reprocessamento simples

- Separação clara de responsabilidades entre armazenamentos

- Escalabilidade futura

# Características Técnicas:

- Arquitetura orientada a eventos

- Particionamento temporal

- Escrita em formato columnar (Parquet)

- Workflow para orquestração

- Controle explícito de tipos

# Idempotência

Os jobs da camada Gold são idempotentes. Cada execução sobrescreve apenas a partição correspondente ao período processado, garantindo consistência caso o job seja reexecutado.

# Processamento Incremental

Os jobs Gold processam apenas as partições necessárias da camada Silver, evitando leitura completa do dataset e reduzindo custo computacional.

# Modelo Medalhão:

## Camada Bronze

A camada Bronze armazena os dados brutos exatamente como retornados pela API pública da CoinGecko.

• Características:

- Formato: JSON

- Particionamento por data (year/month/day)

- Nenhuma transformação aplicada

- Dados imutáveis

A ingestão é realizada por uma função Lambda acionada a cada 1 minuto por EventBridge.

• Responsabilidades da Lambda:

- Consumir API externa

- Criar partição temporal

- Persistir JSON bruto no S3

- Garantir rastreabilidade por timestamp

Esse modelo simula ingestão contínua (stream-like) via polling controlado.

## Camada Silver

• A camada Silver é responsável por:

- Tipagem explícita das colunas

- Renomeação padronizada

- Conversão de timestamps

- Particionamento por data da fonte

- Escrita em formato Parquet

O processamento é realizado por um Glue Job acionado por um Glue Workflow, disparado por evento de criação de objeto no S3.

• Transformações Aplicadas:

- Cast para Double, Integer e Long

- Conversão de last_updated para timestamp

- Criação de coluna processed_at

- Particionamento por year/month/day

O objetivo dessa camada é fornecer dados consistentes e prontos para consumo analítico.

## Camada Gold

• A camada Gold é responsável por:

- Agregações por hora

- Agregações por dia

- Métricas consolidadas

- Base otimizada para BI e dashboards

O processamento é realizado por 2 Glues Jobs distintos, com schedules acionados por hora e por dia, para manter métricas relevantes para análise posterior.

# AWS S3 Buckets Estruturados por Camada

Estruturação do S3 com modelo medalhão:

<img width="696" height="375" alt="image" src="https://github.com/user-attachments/assets/1713ebf7-15d1-4cf8-b969-a5eaee6b33c6" />
![Modelo](images/Modelo_S3.png)

# AWS Lambda 

<img width="1183" height="510" alt="image" src="https://github.com/user-attachments/assets/e12da1f8-36af-4e98-8e22-4e637b073ef1" />

Código de exemplo para extração do arquivo JSON e inserção na camada bronze do S3:

`````

import json
import boto3
import os
from datetime import datetime
import urllib.request

s3 = boto3.client("s3")
BUCKET_NAME = os.environ.get("Digite o nome da variável de ambiente que você salvar no lambda")

# URL da API
API_URL = "https://api.coingecko.com/api/v3/coins/markets?vs_currency=usd" # URL publica da API

def lambda_handler(event, context):
    """
    Lambda disparada pelo EventBridge.
    - Faz requisição à API externa
    - Salva os dados brutos na camada bronze do S3
    """

    try:
        # Chama a API externa
        with urllib.request.urlopen(API_URL) as response:
            payload = json.loads(response.read().decode())

        # Cria partição por data
        now = datetime.utcnow()
        partition_path = now.strftime("year=%Y/month=%m/day=%d")
        file_name = f"crypto_{now.strftime('%Y%m%dT%H%M%S%f')}.json"
        s3_key = f"bronze/crypto/{partition_path}/{file_name}"

        # Serializa e envia para o S3
        body = json.dumps(payload, ensure_ascii = False)
        s3.put_object(
            Bucket = BUCKET_NAME,
            Key = s3_key,
            Body = body.encode("utf-8"),
            ContentType = "application/json"
        )

        return {
            "statusCode": 200,
            "message": f"Arquivo salvo em s3://{BUCKET_NAME}/{s3_key}"
        }

    except Exception as e:
        print('Ocorreu um Erro:', e)
        return {
            "statusCode": 500,
            "error": str(e)
        }
        
`````

## Métricas do Lambda:

<img width="1589" height="686" alt="image" src="https://github.com/user-attachments/assets/57e06845-165f-4d1e-953c-d0e554a02aae" />

## JSON Registrado na camada bronze:

<img width="1858" height="413" alt="image" src="https://github.com/user-attachments/assets/a69ae1ce-5645-4646-8e96-a654dcaa0428" />

## Configurações do EventBridge Schedule:

<img width="1394" height="759" alt="image" src="https://github.com/user-attachments/assets/b97bb478-004e-4730-b376-ecc98a34e245" />


# AWS S3 Events

Configurações de propriedades do bucket bronze:

<img width="1515" height="361" alt="image" src="https://github.com/user-attachments/assets/14d0695a-90e9-48ae-bf7f-1bc88f8fbb66" />

## Event Pattern configurado no EventBridge Rule:

<img width="1546" height="463" alt="image" src="https://github.com/user-attachments/assets/275bf7e9-2d9e-449b-9c19-fe381a683171" />

## Configurações do Target dentro do EventBridge Rule:

<img width="1567" height="321" alt="image" src="https://github.com/user-attachments/assets/b9d66676-0cc4-41a2-bfbb-61735d17de3d" />


# AWS Glue Workflow

Configurações do trigger para a orquestração funcionar corretamente:

<img width="1576" height="588" alt="image" src="https://github.com/user-attachments/assets/27bde67c-9c9f-4507-bb30-6188190bce2d" />

<img width="373" height="295" alt="image" src="https://github.com/user-attachments/assets/88aea68d-505d-455a-beb6-7480a489587e" />


# AWS Glue Job (Silver)

Código utilizado para o funcionamento do processamento do JSON:

`````

import sys
from pyspark.sql.functions import (
    col,
    to_timestamp,
    year,
    month,
    dayofmonth,
    current_timestamp
)
from pyspark.sql.types import DoubleType, IntegerType, LongType, StringType
from awsglue.utils import getResolvedOptions
from awsglue.context import GlueContext
from pyspark.context import SparkContext


# =========================
# Inicialização
# =========================
args = getResolvedOptions(sys.argv, [])

sc = SparkContext()
glueContext = GlueContext(sc)
spark = glueContext.spark_session


# =========================
# Caminhos
# =========================
BRONZE_PATH = "s3://main-bronze-bucket/bronze/crypto/"
SILVER_PATH = "s3://main-silver-bucket/silver/crypto/"


# =========================
# Leitura Bronze
# =========================
df = spark.read.json(BRONZE_PATH)


# =========================
# Transformações (renomeação + cast de tipos)
# =========================
df_transformed = (
    df.select(
        col("id").cast(StringType()).alias("name"),
        col("current_price").cast(DoubleType()).alias("price_usd"),
        col("market_cap").cast(LongType()).alias("market_cap_usd"),
        col("market_cap_rank").cast(IntegerType()).alias("market_rank"),
        col("high_24h").cast(DoubleType()).alias("high_24h_usd"),
        col("low_24h").cast(DoubleType()).alias("low_24h_usd"),
        col("price_change_24h").cast(DoubleType()).alias("price_change_24h_usd"),
        col("price_change_percentage_24h").cast(DoubleType()).alias("price_change_pct_24h"),
        col("ath").cast(DoubleType()).alias("all_time_high_usd"),
        col("ath_change_percentage").cast(DoubleType()).alias("all_time_high_change_pct"),
        col("atl").cast(DoubleType()).alias("all_time_low_usd"),
        col("atl_change_percentage").cast(DoubleType()).alias("all_time_low_change_pct"),
        to_timestamp(
            col("last_updated"),
            "yyyy-MM-dd'T'HH:mm:ss.SSSX"
        ).alias("update_timestamp"),
        current_timestamp().alias("processed_at")
    )
)

# Remove registros inválidos
df_transformed = df_transformed.filter(
    col("update_timestamp").isNotNull()
)


# =========================
# Particionamento por data da fonte
# =========================
df_partitioned = (
    df_transformed
        .withColumn("year", year(col("update_timestamp")))
        .withColumn("month", month(col("update_timestamp")))
        .withColumn("day", dayofmonth(col("update_timestamp")))
)


# =========================
# Escrita Silver em Parquet
# =========================
(
    df_partitioned
        .write
        .mode("append")
        .partitionBy("year", "month", "day")
        .parquet(SILVER_PATH)
)

print("Job finalizado com sucesso.")

`````

## Run do Job rodando com sucesso:

<img width="1557" height="699" alt="image" src="https://github.com/user-attachments/assets/b3857acc-a2d3-4aba-b53c-36c3bb2614e5" />

## Arquivo criado com sucesso na camada silver:

<img width="1851" height="414" alt="image" src="https://github.com/user-attachments/assets/bed00bf3-a514-4611-9ade-1d44987772cd" />

# AWS Glue (Gold - p/hour)

Código utilizado para o processamento da camada prata a cada hora:

`````

import sys
from awsglue.utils import getResolvedOptions
from pyspark.sql.functions import (
    col,
    avg,
    max,
    min,
    stddev,
    year,
    month,
    dayofmonth,
    date_trunc,
    first,
    count,
    input_file_name,
    hour
)
from pyspark.sql.types import DecimalType
from pyspark.sql.window import Window
from pyspark.context import SparkContext
from awsglue.context import GlueContext
from datetime import datetime, timedelta
import boto3
import time

# =========================
# Inicialização
# =========================
args = getResolvedOptions(sys.argv, [])

sc = SparkContext()
glueContext = GlueContext(sc)
spark = glueContext.spark_session

spark.conf.set("spark.sql.sources.partitionOverwriteMode", "dynamic")

# =========================
# Caminhos
# =========================
SILVER_PATH = "s3://main-silver-bucket/silver/crypto"
GOLD_PATH = "s3://main-gold-bucket/gold/crypto/hourly"

# =========================
# Data, ano, mês, dias e horas atuais
# =========================
target_time = datetime.utcnow() - timedelta(hours = 1)

target_year = target_time.year
target_month = target_time.month
target_day = target_time.day
target_hour = target_time.hour

# =========================
# Leitura Silver filtrando por dia atual (para não gerar lentidão nem custo lendo todo o silver)
# =========================
silver_path_today = (
    f"{SILVER_PATH}/year={target_year}/"
    f"month={target_month}/"
    f"day={target_day}/"
)

df = spark.read.parquet(silver_path_today)

# =========================
# Truncar e filtrar apenas hora atual
# =========================
df = df.filter(
    hour(col("update_timestamp")) == target_hour
)

df = df.withColumn(
    "hour_timestamp",
    date_trunc("hour", col("update_timestamp"))
)

# =========================
# Window para OHLC real
# =========================
window_spec_open = Window.partitionBy("name", "hour_timestamp") \
                          .orderBy(col("update_timestamp").asc())

window_spec_close = Window.partitionBy("name", "hour_timestamp") \
                           .orderBy(col("update_timestamp").desc())

df = df.withColumn("price_open_hour",
                   first("price_usd").over(window_spec_open))

df = df.withColumn("price_close_hour",
                   first("price_usd").over(window_spec_close))
                   
# =========================
# Agregação por ativo + hora
# =========================
df_gold = (
    df.groupBy("name", "hour_timestamp")
    .agg(
         max("price_open_hour").alias("price_open_hour"),
         max("price_close_hour").alias("price_close_hour"),
         max("price_usd").alias("price_high_hour"),
         min("price_usd").alias("price_low_hour"),
         avg("price_usd").alias("price_avg_hour"),
         stddev("price_usd").alias("price_volatility_hour"),
         avg("market_cap_usd").alias("market_cap_avg_hour"),
         max("market_cap_usd").alias("market_cap_max_hour"),
         min("market_cap_usd").alias("market_cap_min_hour"),
         avg("price_change_24h_usd").alias("avg_price_change_24h_usd"),
         avg("price_change_pct_24h").alias("avg_price_change_pct_24h"),
         count("*").alias("records_in_hour")
    )
)

df_gold = df_gold.select(
    col("name"),
    col("hour_timestamp"),

    col("price_open_hour").cast(DecimalType(20,10)),
    col("price_close_hour").cast(DecimalType(20,10)),
    col("price_high_hour").cast(DecimalType(20,10)),
    col("price_low_hour").cast(DecimalType(20,10)),
    col("price_avg_hour").cast(DecimalType(20,10)),
    col("price_volatility_hour").cast(DecimalType(20,10)),

    col("market_cap_avg_hour").cast(DecimalType(20,2)),
    col("market_cap_max_hour"),
    col("market_cap_min_hour"),

    col("avg_price_change_24h_usd").cast(DecimalType(20,10)),
    col("avg_price_change_pct_24h").cast(DecimalType(10,6)),

    col("records_in_hour")
)

# =========================
# Particionamento por data da fonte
# =========================
df_gold = (
    df_gold
        .withColumn("year", year(col("hour_timestamp")))
        .withColumn("month", month(col("hour_timestamp")))
        .withColumn("day", dayofmonth(col("hour_timestamp")))
        .withColumn("hour", hour(col("hour_timestamp")))
)

# =========================
# Escrita Gold
# =========================
(
    df_gold
        .write
        .mode("overwrite")
        .partitionBy("year", "month", "day", "hour")
        .parquet(GOLD_PATH)
)

print("Job finalizado com sucesso.")

`````

## Run do Job rodando com sucesso:

<img width="1561" height="697" alt="image" src="https://github.com/user-attachments/assets/43a3559e-8d41-403b-bea3-8b6e5a64a4ea" />


## Arquivo criado com sucesso na camada gold:

<img width="1869" height="421" alt="image" src="https://github.com/user-attachments/assets/f4b5e2bd-8be7-41c4-ae04-b24c527c5ef0" />

# AWS Glue (Gold p/day)

Código utilizado para o processamento da camada prata a cada dia:

`````

import sys
from awsglue.utils import getResolvedOptions
from pyspark.sql.functions import (
    col,
    avg,
    max,
    min,
    stddev,
    year,
    month,
    dayofmonth,
    date_trunc,
    first,
    count
)
from pyspark.sql.types import DecimalType
from pyspark.sql.window import Window
from pyspark.context import SparkContext
from awsglue.context import GlueContext
from datetime import datetime, timedelta
import boto3
import time

# =========================
# Inicialização
# =========================
args = getResolvedOptions(sys.argv, [])

sc = SparkContext()
glueContext = GlueContext(sc)
spark = glueContext.spark_session

spark.conf.set("spark.sql.sources.partitionOverwriteMode", "dynamic")

# =========================
# Caminhos
# =========================
SILVER_PATH = "s3://main-silver-bucket/silver/crypto/"
GOLD_PATH = "s3://main-gold-bucket/gold/crypto/daily"

# =========================
# Dia fechado anterior
# =========================
target_date = (datetime.utcnow() - timedelta(days = 1)).date()

start_day = datetime.combine(target_date, datetime.min.time())
end_day = start_day + timedelta(days = 1)

target_year = start_day.year
target_month = start_day.month
target_day = start_day.day

# =========================
# Leitura apenas da partição do dia
# =========================
silver_path_target = (
    f"{SILVER_PATH}/year={target_year}/"
    f"month={target_month}/"
    f"day={target_day}/"
)

df = spark.read.parquet(silver_path_target)

# =========================
# Filtro por intervalo do dia (robusto)
# =========================
df = df.filter(
    (col("update_timestamp") >= start_day) &
    (col("update_timestamp") < end_day)
)

df = df.withColumn(
    "day_timestamp",
    date_trunc("day", col("update_timestamp"))
)

# =========================
# Window para OHLC diário real
# =========================
window_spec_open = Window.partitionBy("name", "day_timestamp") \
                         .orderBy(col("update_timestamp").asc())

window_spec_close = Window.partitionBy("name", "day_timestamp") \
                          .orderBy(col("update_timestamp").desc())

df = df.withColumn(
    "price_open_day",
    first("price_usd").over(window_spec_open)
)

df = df.withColumn(
    "price_close_day",
    first("price_usd").over(window_spec_close)
)

# =========================
# Agregação diária
# =========================
df_gold = (
    df.groupBy("name", "day_timestamp")
      .agg(
          max("price_open_day").alias("price_open_day"),
          max("price_close_day").alias("price_close_day"),
          max("price_usd").alias("price_high_day"),
          min("price_usd").alias("price_low_day"),
          avg("price_usd").alias("price_avg_day"),
          stddev("price_usd").alias("price_volatility_day"),
          avg("market_cap_usd").alias("market_cap_avg_day"),
          max("market_cap_usd").alias("market_cap_max_day"),
          min("market_cap_usd").alias("market_cap_min_day"),
          avg("price_change_24h_usd").alias("avg_price_change_24h_usd"),
          avg("price_change_pct_24h").alias("avg_price_change_pct_24h"),
          count("*").alias("records_in_day")
      )
)

df_gold = df_gold.select(
    col("name"),
    col("day_timestamp"),

    col("price_open_day").cast(DecimalType(20,10)),
    col("price_close_day").cast(DecimalType(20,10)),
    col("price_high_day").cast(DecimalType(20,10)),
    col("price_low_day").cast(DecimalType(20,10)),
    col("price_avg_day").cast(DecimalType(20,10)),
    col("price_volatility_day").cast(DecimalType(20,10)),

    col("market_cap_avg_day").cast(DecimalType(20,2)),
    col("market_cap_max_day"),
    col("market_cap_min_day"),

    col("avg_price_change_24h_usd").cast(DecimalType(20,10)),
    col("avg_price_change_pct_24h").cast(DecimalType(10,6)),

    col("records_in_day")
)

# =========================
# Particionamento
# =========================
df_gold = (
    df_gold
        .withColumn("year", year(col("day_timestamp")))
        .withColumn("month", month(col("day_timestamp")))
        .withColumn("day", dayofmonth(col("day_timestamp")))
)

# =========================
# Escrita idempotente
# =========================
(
    df_gold
        .write
        .mode("overwrite")
        .partitionBy("year", "month", "day")
        .parquet(GOLD_PATH)
)

print('Job diário finalizado com sucesso.')

`````

## Run do Job rodando com sucesso:

<img width="1561" height="697" alt="image" src="https://github.com/user-attachments/assets/43a3559e-8d41-403b-bea3-8b6e5a64a4ea" />


## Arquivo criado com sucesso na camada gold:

<img width="1866" height="421" alt="image" src="https://github.com/user-attachments/assets/b6db2b14-9f3a-415f-85d9-61af86c6c6a4" />

# AWS Athena + AWS Glue Data Catalog

## Código utilizado para criar o schema da camada gold (por hora)

`````

CREATE EXTERNAL TABLE hourly (
  name string,
  hour_timestamp timestamp,
  price_open_hour decimal(20,10),
  price_close_hour decimal(20,10),
  price_high_hour decimal(20,10),
  price_low_hour decimal(20,10),
  price_avg_hour decimal(20,10),
  price_volatility_hour decimal(20,10),
  market_cap_avg_hour decimal(20,2),
  market_cap_max_hour bigint,
  market_cap_min_hour bigint,
  avg_price_change_24h_usd decimal(20,10),
  avg_price_change_pct_24h decimal(10,6),
  records_in_hour bigint
)
PARTITIONED BY (
  year int,
  month int,
  day int,
  hour int
)
STORED AS PARQUET
LOCATION 's3://main-gold-bucket/gold/crypto/hourly/';

`````

Partition projection configurado via código:

`````

ALTER TABLE hourly
SET TBLPROPERTIES (
  'projection.enabled'='true',

  'projection.year.type'='integer',
  'projection.year.range'='2024,2030',

  'projection.month.type'='integer',
  'projection.month.range'='1,12',

  'projection.day.type'='integer',
  'projection.day.range'='1,31',

  'projection.hour.type'='integer',
  'projection.hour.range'='0,23',

  'storage.location.template'='s3://main-gold-bucket/gold/crypto/hourly/year=${year}/month=${month}/day=${day}/hour=${hour}/'
);

`````

## Código utilizado para criar o schema da camada gold (por dia)

`````

CREATE EXTERNAL TABLE daily (
  name string,
  day_timestamp timestamp,
  price_open_day decimal(20,10),
  price_close_day decimal(20,10),
  price_high_day decimal(20,10),
  price_low_day decimal(20,10),
  price_avg_day decimal(20,10),
  price_volatility_day decimal(20,10),
  market_cap_avg_day decimal(20,2),
  market_cap_max_day bigint,
  market_cap_min_day bigint,
  avg_price_change_24h_usd decimal(20,10),
  avg_price_change_pct_24h decimal(10,6),
  records_in_day bigint
)
PARTITIONED BY (
  year int,
  month int,
  day int
)
STORED AS PARQUET
LOCATION 's3://main-gold-bucket/gold/crypto/daily/';

`````

Partition projection configurado via código:

`````

ALTER TABLE daily
SET TBLPROPERTIES (
  'projection.enabled'='true',

  'projection.year.type'='integer',
  'projection.year.range'='2024,2030',

  'projection.month.type'='integer',
  'projection.month.range'='1,12',

  'projection.day.type'='integer',
  'projection.day.range'='1,31',

  'storage.location.template'='s3://main-gold-bucket/gold/crypto/daily/year=${year}/month=${month}/day=${day}/'
);

`````

## DataBase criado dentro do Data Catalog:

<img width="1583" height="201" alt="image" src="https://github.com/user-attachments/assets/a5e3a9eb-0b82-475f-8ceb-d492f43bf92e" />

## Tabelas criadas com sucesso após o run do Athena:

<img width="1860" height="423" alt="image" src="https://github.com/user-attachments/assets/0b69b53b-a131-4151-aa07-4315593826e6" />

## Schemas padronizados dentro da tabela daily criada:

<img width="1558" height="744" alt="image" src="https://github.com/user-attachments/assets/f4f28dd7-0021-4065-bf55-244d428172d2" />

## Schemas padronizados dentro da tabela hourly criada:

<img width="1566" height="712" alt="image" src="https://github.com/user-attachments/assets/4a06c415-af90-4b46-940c-f5432ba81a19" />

## SELECT utilizado para testar consulta da tabela hourly no Athena:

Observação: Os dados de ambas as querys estão iguais em ambas as querys por conta do range utilizado para a camada prata.

`````

SELECT *
FROM hourly
WHERE year = 2026 AND month = 3 AND day = 4 AND hour = 22;

`````

Resultado da Query utilizada:

<img width="1534" height="816" alt="image" src="https://github.com/user-attachments/assets/0bc3bfc8-08d9-45d5-ba8e-a8295e412cf4" />

<img width="1541" height="863" alt="image" src="https://github.com/user-attachments/assets/7c87fbac-272e-4b8e-9d9e-db2ee2b6ab36" />

## SELECT utilizado para testar consulta da tabela daily no Athena:

`````

SELECT *
FROM daily
WHERE year = 2026 AND month = 3 AND day = 11;

`````

Resultado da Query utilizada:

<img width="1608" height="725" alt="image" src="https://github.com/user-attachments/assets/319b23a3-ac1a-45a5-8f12-785ca73e83dd" />

<img width="1569" height="729" alt="image" src="https://github.com/user-attachments/assets/012364d0-9f89-431c-b72f-da7a0af2705e" />


# Fim do Projeto
