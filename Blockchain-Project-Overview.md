# Blockchain Analytics Platform

I bring hands-on experience building data infrastructures and pipelines for blockchain analytics, having contributed to both a blockchain fraud analytics firm and an AI-based DEX platform.

Below is a brief overview of my experience and skills:

## 1. Comprehensive Blockchain Data Experience
* **Multi-Chain Coverage**: Worked extensively with data from various blockchains, including Ethereum, BNB Smart Chain (BSC), Polygon, Avalanche, Arbitrum, Optimism, Tron, and Base.
* **Raw Data Handling**: Built pipelines that ingest raw transactions, logs, traces, and blocks, ensuring data integrity and high availability for downstream applications.

## 2. Enriched & Derived Data Pipelines
* **Enriched Tables**: Developed data transformation workflows that combine raw logs, transaction details, ERC token data, and price feeds to create DEX trade datasets and other high-value tables, updated on an hourly basis.
* **Data Harmonization**: Created schemas to unify data structures across multiple protocols and blockchains, streamlining analytics and reporting.

## 3. Bridging & Cross-Chain Analytics
* **Asset Movement Tracking**: Implemented bridge-monitoring solutions that track asset flows across major L1 and L2 networks, providing transparency into cross-chain liquidity.
* **Stablecoin Monitoring**: Designed macro metrics to granular transaction views for stablecoin movement, enabling real-time risk assessment and market insight.

## Project Overview
In this project, we built a robust multi-chain data pipeline and analytics layer for DEX trades across several blockchain networks (Avalanche, Arbitrum, Base, BSC, Ethereum, Optimism, Polygon, Tron). Below is a high-level summary of the core components and their purposes:

### 1. Unified DEX Trades Table
* **Purpose**: Consolidate trades from multiple chains into a single table, DEX_ALL_TRADES.
* **Implementation**:
   * A CREATE OR REPLACE TABLE statement unions each chain's DEX.TRADES table into a standardized schema.
   * A corresponding task UPDATE_DEX_ALL_TRADES uses streams on each chain's trades table to incrementally update DEX_ALL_TRADES on an hourly schedule.

### 2. Data Streaming & Incremental Updates
* **Purpose**: Ingest new on-chain DEX trades data as it arrives.
* **Implementation**:
   * Streams (AVALANCHE_DEX_TRADES_STREAM, etc.) capture table changes in real-time.
   * The UPDATE_DEX_ALL_TRADES task runs hourly to merge fresh trades into the unified table without reprocessing historical data.

### 3. Token PnL Calculation per Wallet
* **Purpose**: Compute individual wallet-level profit and loss for each token traded.
* **Implementation**:
   * The refresh_token_pnl_per_wallet_task runs daily to generate token_pnl_per_wallet, which calculates metrics like total tokens bought/sold, total USD invested/received, and realized PnL (profit/loss).

### 4. Aggregated PnL per Wallet
* **Purpose**: Provide an overall profit/loss summary across all tokens a wallet holds.
* **Implementation**:
   * The refresh_overall_pnl_per_wallet_task aggregates the metrics from token_pnl_per_wallet, summing up realized PnL and cost basis to produce a comprehensive view of each wallet's trading performance.

### 5. Top 100 Traders per Token
* **Purpose**: Identify the most profitable traders for each token (by realized profit).
* **Implementation**:
   * The refresh_top_100_traders_per_token_task filters out non-user addresses (contract addresses, labeled addresses) and ranks traders based on realized profit.
   * It stores the top 100 for each token in top_100_traders_per_token.

### 6. Top Traders Aggregation & Trading Activity
* **Purpose**: Further analyze the trading patterns of the most profitable traders (e.g., how much they trade, token categories, market cap data).
* **Implementation**:
   * The REFRESH_TOP_TRADERS_TASK creates a top_traders_table that enriches wallet data with Coingecko stats (market cap, category) and consolidates recent trading activity over the last month.
   * It also categorizes tokens (e.g., Native, Stablecoins, etc.) and aggregates trades by time periods (daily, weekly, monthly).

Overall, the project establishes a scalable data flow for capturing, unifying, and analyzing DEX trades across multiple blockchains. The system supports incremental updates, automated daily tasks, and advanced wallet-level or token-level profit analysis, enabling downstream stakeholders to gain real-time and historical insights into trading behavior and market activity.

## Technology Stack

### Data Infrastructure & Processing
- **SnowFlake**: Primary data warehouse for analytics and transformations
  - Handles complex SQL transformations
  - Manages incremental data loading
  - Streams processing for real-time updates

- **Clickhouse**: High-performance analytics database
  - Real-time data ingestion
  - Fast aggregations on large datasets
  - Efficient storage of time-series data

- **TimeScaleDB**: Time-series database for blockchain data
  - Historical transaction storage
  - Time-based aggregations
  - Performance optimization for temporal queries

### Cloud & Infrastructure
- **AWS/GCP**: Cloud infrastructure
  - Scalable compute resources
  - Managed services integration
  - High availability setup

- **Kubernetes**: Container orchestration
  - Microservices deployment
  - Scaling and resource management
  - Service reliability

### Query Engines
- **BigQuery/Athena**: Data analysis
  - Large-scale data processing
  - Ad-hoc analytics
  - Cost-effective querying

### Development
- **Python**: Core development
  - ETL pipeline implementation
  - Data processing scripts
