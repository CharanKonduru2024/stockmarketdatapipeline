# Real-Time Stock Market Data Pipeline

End-to-end data engineering project that ingests stock market data (historical + simulated real-time), processes it with Apache Spark, stores it in an S3-compatible data lake (MinIO), orchestrates workflows with Apache Airflow, and loads curated metrics into Snowflake for analytics.

The pipeline is fully containerized with Docker Compose and uses Kafka as the event backbone.

---

## Features

- **Dual Ingestion Paths**
  - **Batch / Historical**: Uses `yfinance` to fetch OHLCV history for multiple tickers (AAPL, MSFT, GOOGL, AMZN, META, TSLA, NVDA, …) and publishes records to Kafka.
  - **Real-Time (Simulated)**: Generates synthetic tick-level price and volume updates and streams them to Kafka in near real-time.

- **Kafka Streaming Layer**
  - Separate **producers** for historical and real-time topics.
  - **Consumers** read from Kafka and land data into MinIO in a partitioned layout, ready for Spark jobs.

- **Data Lake on MinIO (S3-compatible)**
  - Organized, time-partitioned buckets for:
    - `raw/historical/year=YYYY/month=MM/day=DD/`
    - `raw/realtime/year=YYYY/month=MM/day=DD/hour=HH/`
    - `processed/historical/date=YYYY-MM-DD/`
    - `processed/realtime/` (streaming outputs + checkpoints)

- **Batch Analytics with Spark**
  - `spark_batch_processor.py` reads raw historical data from MinIO and computes:
    - `daily_open`, `daily_high`, `daily_low`, `daily_close`, `daily_volume`
    - `% daily_change` vs. open
  - Writes curated daily metrics back to MinIO under `processed/historical/`.

- **Real-Time Analytics with Spark Structured Streaming**
  - `spark_stream_processor.py` consumes streaming data from MinIO and calculates:
    - 15-minute and 1-hour moving averages
    - 15-minute and 1-hour volatility (std dev)
    - 15-minute and 1-hour aggregated volume
  - Uses sliding windows and `foreachBatch` to write micro-batch outputs to MinIO.

- **Snowflake Warehouse Integration**
  - `load_to_snowflake.py` reads processed historical data from MinIO.
  - Creates a Snowflake table if it doesn’t exist.
  - Performs **incremental upserts** using a `MERGE` from a temporary stage table on `(symbol, date)`.

- **Workflow Orchestration with Airflow**
  - DAG: `stock_market_batch_dag.py` orchestrates:
    1. Fetch historical data and publish to Kafka
    2. Consume batch data into MinIO
    3. Run Spark batch processing
    4. Load curated data into Snowflake
    5. Log completion
  - Includes a MinIO data-existence check step via `check_minio_file.py`.

- **Containerized Infra**
  - One-command local stack using Docker Compose:
    - Kafka + Zookeeper
    - MinIO
    - Spark master/worker
    - Postgres (for Airflow)
    - Airflow webserver + scheduler

---

## Architecture

```mermaid
flowchart LR
  subgraph Ingestion
    YF[yfinance] --> BP[BatchProducer]
    SIM[PriceGenerator] --> SP[StreamProducer]
  end

  BP --> KB[KafkaBatchTopic]
  SP --> KR[KafkaRealtimeTopic]

  subgraph MinIO
    CBatch[BatchConsumer] --> RAW[RawHistorical]
    CReal[RealtimeConsumer] --> RAWRT[RawRealtime]
  end

  KB --> CBatch
  KR --> CReal

  subgraph Spark
    RAW --> SB[BatchJob]
    RAWRT --> SS[StreamingJob]
  end

  SB --> PROCH[ProcessedHistorical]
  SS --> PROCRT[ProcessedRealtime]

  PROCH --> SFLoader[SnowflakeLoader]
  SFLoader --> SF[SnowflakeTable]


