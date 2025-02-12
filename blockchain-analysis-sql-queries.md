# Advanced SQL Queries and Analyses for Blockchain Data

## Table of Contents
1. [Introduction](#introduction)
2. [Dataset Overview](#dataset-overview)
3. [Advanced SQL Queries and Analyses](#advanced-sql-queries-and-analyses)

   3.1 [Detecting Circular Trading Patterns](#31-detecting-circular-trading-patterns)

   3.2 [Identifying Dormant Wallets Suddenly Receiving Large Funds](#32-identifying-dormant-wallets-suddenly-receiving-large-funds)

   3.3 [Cross-Chain Arbitrage Opportunities Detection](#33-cross-chain-arbitrage-opportunities-detection)

   3.4 [High-Frequency Trading Wallets Detection](#34-high-frequency-trading-wallets-detection)

   3.5 [Analyzing Fee Consumption Patterns](#35-analyzing-fee-consumption-patterns)

   3.6 [Token Holding Period Analysis](#36-token-holding-period-analysis)

   3.7 [Correlation Between Token Prices and Transaction Volumes](#37-correlation-between-token-prices-and-transaction-volumes)

   3.8 [Detection of Potential Pump and Dump Schemes](#38-detection-of-potential-pump-and-dump-schemes)

   3.9 [Analysis of Counterparty Risk Exposure](#39-analysis-of-counterparty-risk-exposure)

   3.10 [Predictive Modeling for Transaction Amounts](#310-predictive-modeling-for-transaction-amounts)

4. [Implementing and Extending the Queries](#implementing-and-extending-the-queries)
5. [Potential Applications](#potential-applications)

## 1. Introduction

This guide provides advanced SQL queries designed to perform in-depth analyses on blockchain transaction data. These queries aim to uncover unique insights that are not commonly explored by other companies. Each query includes an explanation of its purpose and the valuable information it can reveal.

## 2. Dataset Overview

Before diving into the queries, let's briefly understand the datasets:

### 2.1 WALLET_COUNTERPARTY_SUMMARY

Columns:
- CHAIN
- WALLET_ADDRESS
- COUNTERPARTY
- DIRECTION
- FIRST_SEEN_DATE
- LAST_SEEN_DATE
- TOTAL_USD_AMOUNT
- TRANSACTION_COUNT
- AVERAGE_USD_AMOUNT
- MIN_USD_AMOUNT
- MAX_USD_AMOUNT
- WALLET_NAME
- WALLET_PROJECT
- WALLET_CATEGORY
- COUNTERPARTY_NAME
- COUNTERPARTY_PROJECT
- COUNTERPARTY_CATEGORY
- WALLET_LIST_TYPE
- WALLET_FLAG
- COUNTERPARTY_LIST_TYPE
- COUNTERPARTY_FLAG
- FLAG_WALLET_NON_WHITELIST_BLACKLIST_COUNTERPARTY
- FLAG_WALLET_WHITELIST_BLACKLIST_COUNTERPARTY
- FLAG_WALLET_WHITELIST_COUNTERPARTY_INTERACTED_WITH_BLACKLIST

### 2.2 token_transfers

Columns:
- CHAIN
- WALLET_ADDRESS
- DIRECTION
- FROM_ADDRESS
- TO_ADDRESS
- TOKEN_ADDRESS
- TOKEN_NAME
- TOKEN_SYMBOL
- AMOUNT
- USD_AMOUNT
- TRANSACTION_HASH
- BLOCK_TIMESTAMP
- BLOCK_NUMBER
- BLOCK_HASH
- UNIQUE_ID

## 3. Advanced SQL Queries and Analyses

### 3.1 Detecting Circular Trading Patterns

**Objective:** Identify wallets engaged in circular trading, where funds move through a series of wallets and return to the original wallet, potentially indicating wash trading or money laundering.

**SQL Query:**

```sql
WITH first_transfers AS (
    SELECT
        t1.WALLET_ADDRESS AS wallet_a,
        t1.COUNTERPARTY AS wallet_b,
        t1.FIRST_SEEN_DATE AS transfer_time
    FROM WALLET_COUNTERPARTY_SUMMARY t1
    WHERE t1.DIRECTION = 'sent'
),
second_transfers AS (
    SELECT
        t2.WALLET_ADDRESS AS wallet_b,
        t2.COUNTERPARTY AS wallet_c,
        t2.FIRST_SEEN_DATE AS transfer_time
    FROM WALLET_COUNTERPARTY_SUMMARY t2
    WHERE t2.DIRECTION = 'sent'
),
third_transfers AS (
    SELECT
        t3.WALLET_ADDRESS AS wallet_c,
        t3.COUNTERPARTY AS wallet_a,
        t3.FIRST_SEEN_DATE AS transfer_time
    FROM WALLET_COUNTERPARTY_SUMMARY t3
    WHERE t3.DIRECTION = 'sent'
),
circular_trades AS (
    SELECT
        ft.wallet_a,
        ft.wallet_b,
        st.wallet_c,
        tt.wallet_a AS wallet_return,
        ft.transfer_time AS time_a_to_b,
        st.transfer_time AS time_b_to_c,
        tt.transfer_time AS time_c_to_a
    FROM first_transfers ft
    JOIN second_transfers st ON ft.wallet_b = st.wallet_b
    JOIN third_transfers tt ON st.wallet_c = tt.wallet_c AND ft.wallet_a = tt.wallet_a
    WHERE tt.transfer_time > st.transfer_time AND st.transfer_time > ft.transfer_time
)
SELECT *
FROM circular_trades
ORDER BY time_c_to_a DESC;
```

**Explanation:**
- Purpose: Identifies sequences where funds move from wallet_a to wallet_b, then to wallet_c, and back to wallet_a.
- Unique Insight: Circular trading can be indicative of attempts to manipulate market volumes or obscure the origin of funds.

### 3.2 Identifying Dormant Wallets Suddenly Receiving Large Funds

**Objective:** Find wallets that were inactive for an extended period and then received large transactions, which may indicate compromised wallets or preparation for significant market moves.

**SQL Query:**

```sql
WITH wallet_activity AS (
    SELECT
        WALLET_ADDRESS,
        MAX(LAST_SEEN_DATE) AS last_seen,
        SUM(TOTAL_USD_AMOUNT) AS total_received
    FROM WALLET_COUNTERPARTY_SUMMARY
    WHERE DIRECTION = 'received'
    GROUP BY WALLET_ADDRESS
),
dormant_wallets AS (
    SELECT
        wa.WALLET_ADDRESS,
        wa.last_seen,
        DATEDIFF(day, wa.last_seen, CURRENT_DATE) AS days_inactive,
        wa.total_received
    FROM wallet_activity wa
    WHERE DATEDIFF(day, wa.last_seen, CURRENT_DATE) > 180 -- Inactive for over 6 months
),
recent_large_receipts AS (
    SELECT
        tt.WALLET_ADDRESS,
        tt.BLOCK_TIMESTAMP,
        tt.USD_AMOUNT
    FROM token_transfers tt
    JOIN dormant_wallets dw ON tt.WALLET_ADDRESS = dw.WALLET_ADDRESS
    WHERE tt.DIRECTION = 'received' AND tt.BLOCK_TIMESTAMP > dw.last_seen
    ORDER BY tt.USD_AMOUNT DESC
)
SELECT *
FROM recent_large_receipts
WHERE USD_AMOUNT > 10000 -- Threshold for large transaction
ORDER BY USD_AMOUNT DESC;
```

**Explanation:**
- Purpose: Detects dormant wallets that have recently become active by receiving significant funds.
- Unique Insight: Such activity may warrant further investigation, as it could signal security breaches or preparation for large-scale transactions.

### 3.3 Cross-Chain Arbitrage Opportunities Detection

**Objective:** Identify wallets that are potentially engaging in cross-chain arbitrage by moving funds between different blockchains within short timeframes.

**SQL Query:**

```sql
WITH cross_chain_transfers AS (
    SELECT
        t1.WALLET_ADDRESS,
        t1.CHAIN AS chain_from,
        t1.BLOCK_TIMESTAMP AS time_from,
        t2.CHAIN AS chain_to,
        t2.BLOCK_TIMESTAMP AS time_to,
        t1.USD_AMOUNT
    FROM token_transfers t1
    JOIN token_transfers t2 ON t1.WALLET_ADDRESS = t2.WALLET_ADDRESS
    WHERE t1.CHAIN != t2.CHAIN
      AND t1.DIRECTION = 'sent'
      AND t2.DIRECTION = 'received'
      AND t2.BLOCK_TIMESTAMP BETWEEN t1.BLOCK_TIMESTAMP AND t1.BLOCK_TIMESTAMP + INTERVAL '10 MINUTE'
)
SELECT
    *,
    EXTRACT(EPOCH FROM (time_to - time_from)) / 60 AS minutes_between
FROM cross_chain_transfers
ORDER BY minutes_between ASC, USD_AMOUNT DESC;
```

**Explanation:**
- Purpose: Finds instances where the same wallet sends funds on one chain and receives on another shortly after.
- Unique Insight: Rapid movement of funds across chains can indicate arbitrage activities, taking advantage of price differences between markets.

### 3.4 High-Frequency Trading Wallets Detection

**Objective:** Identify wallets that perform a high number of transactions within short periods, characteristic of algorithmic or high-frequency trading bots.

**SQL Query:**

```sql
WITH transaction_counts AS (
    SELECT
        WALLET_ADDRESS,
        DATE_TRUNC('hour', BLOCK_TIMESTAMP) AS hour_block,
        COUNT(*) AS tx_count
    FROM token_transfers
    GROUP BY WALLET_ADDRESS, hour_block
),
high_frequency_wallets AS (
    SELECT
        WALLET_ADDRESS,
        hour_block,
        tx_count
    FROM transaction_counts
    WHERE tx_count > 50 -- Threshold for high-frequency activity
)
SELECT
    hfw.WALLET_ADDRESS,
    hfw.hour_block,
    hfw.tx_count,
    wcs.WALLET_CATEGORY,
    wcs.WALLET_NAME
FROM high_frequency_wallets hfw
LEFT JOIN WALLET_COUNTERPARTY_SUMMARY wcs ON hfw.WALLET_ADDRESS = wcs.WALLET_ADDRESS
ORDER BY hfw.tx_count DESC;
```

**Explanation:**
- Purpose: Identifies wallets with unusually high transaction counts within an hour.
- Unique Insight: High-frequency trading wallets can impact market liquidity and volatility; understanding their activity is valuable for market analysis.

### 3.5 Analyzing Fee Consumption Patterns

**Objective:** Determine which wallets are incurring the highest transaction fees, possibly indicating inefficiencies or heavy usage.

**SQL Query:**

```sql
WITH fee_analysis AS (
    SELECT
        WALLET_ADDRESS,
        SUM(TRANSACTION_FEE) AS total_fees,
        COUNT(*) AS tx_count,
        AVG(TRANSACTION_FEE) AS avg_fee
    FROM token_transfers
    GROUP BY WALLET_ADDRESS
)
SELECT
    fa.WALLET_ADDRESS,
    fa.total_fees,
    fa.tx_count,
    fa.avg_fee,
    wcs.WALLET_NAME,
    wcs.WALLET_CATEGORY
FROM fee_analysis fa
LEFT JOIN WALLET_COUNTERPARTY_SUMMARY wcs ON fa.WALLET_ADDRESS = wcs.WALLET_ADDRESS
ORDER BY fa.total_fees DESC
LIMIT 100;
```

**Explanation:**
- Purpose: Identifies wallets that spend the most on transaction fees.
- Unique Insight: High fees may indicate frequent use of the network during peak times or inefficient transaction practices.

### 3.6 Token Holding Period Analysis

**Objective:** Calculate the average holding period of tokens by wallets, revealing investment behaviors such as hodling or short-term trading.

**SQL Query:**

```sql
WITH received_tokens AS (
    SELECT
        WALLET_ADDRESS,
        TOKEN_SYMBOL,
        BLOCK_TIMESTAMP AS received_time,
        USD_AMOUNT AS received_amount,
        TRANSACTION_HASH AS received_tx
    FROM token_transfers
    WHERE DIRECTION = 'received'
),
sent_tokens AS (
    SELECT
        WALLET_ADDRESS,
        TOKEN_SYMBOL,
        BLOCK_TIMESTAMP AS sent_time,
        USD_AMOUNT AS sent_amount,
        TRANSACTION_HASH AS sent_tx
    FROM token_transfers
    WHERE DIRECTION = 'sent'
),
holding_periods AS (
    SELECT
        rt.WALLET_ADDRESS,
        rt.TOKEN_SYMBOL,
        rt.received_time,
        st.sent_time,
        EXTRACT(EPOCH FROM (st.sent_time - rt.received_time)) / (3600 * 24) AS holding_days,
        rt.received_amount,
        st.sent_amount
    FROM received_tokens rt
    JOIN sent_tokens st ON rt.WALLET_ADDRESS = st.WALLET_ADDRESS AND rt.TOKEN_SYMBOL = st.TOKEN_SYMBOL
    WHERE st.sent_time > rt.received_time
)
SELECT
    WALLET_ADDRESS,
    TOKEN_SYMBOL,
    AVG(holding_days) AS average_holding_days,
    COUNT(*) AS transactions
FROM holding_periods
GROUP BY WALLET_ADDRESS, TOKEN_SYMBOL
ORDER BY average_holding_days ASC
LIMIT 100;
```

**Explanation:**
- Purpose: Measures how long wallets hold onto tokens before selling.
- Unique Insight: Short holding periods may indicate speculative trading, while long periods suggest investment confidence.

### 3.7 Correlation Between Token Prices and Transaction Volumes

**Objective:** Analyze whether there's a correlation between token price movements and transaction volumes, potentially indicating causation.

**SQL Query:**

```sql
WITH daily_transactions AS (
    SELECT
        TOKEN_SYMBOL,
        DATE(BLOCK_TIMESTAMP) AS date,
        COUNT(*) AS tx_count,
        SUM(USD_AMOUNT) AS total_volume
    FROM token_transfers
    GROUP BY TOKEN_SYMBOL, date
),
daily_prices AS (
    SELECT
        TOKEN_SYMBOL,
        DATE(TIMESTAMP) AS date,
        AVG(PRICE) AS avg_price
    FROM token_prices
    GROUP BY TOKEN_SYMBOL, date
),
combined_data AS (
    SELECT
        dt.TOKEN_SYMBOL,
        dt.date,
        dt.tx_count,
        dt.total_volume,
        dp.avg_price
    FROM daily_transactions dt
    JOIN daily_prices dp ON dt.TOKEN_SYMBOL = dp.TOKEN_SYMBOL AND dt.date = dp.date
)
SELECT
    TOKEN_SYMBOL,
    CORR(tx_count, avg_price) AS correlation_tx_price,
    CORR(total_volume, avg_price) AS correlation_volume_price
FROM combined_data
GROUP BY TOKEN_SYMBOL
ORDER BY correlation_volume_price DESC;
```

**Explanation:**
- Purpose: Computes the statistical correlation between transaction metrics and token prices.
- Unique Insight: High correlation suggests that transaction activity may be influencing prices or vice versa.

### 3.8 Detection of Potential Pump and Dump Schemes (continued)

**SQL Query (continued):**

```sql
suspicious_activity AS (
    SELECT
        ta.TOKEN_SYMBOL,
        ta.hour_block,
        ta.tx_count,
        ta.total_volume,
        pc.price_change_percent
    FROM token_activity ta
    JOIN price_changes pc ON ta.TOKEN_SYMBOL = pc.TOKEN_SYMBOL AND ta.hour_block = DATE_TRUNC('hour', pc.price_time)
    WHERE pc.price_change_percent > 20 -- Sudden price increase over 20%
)
SELECT
    TOKEN_SYMBOL,
    hour_block,
    tx_count,
    total_volume,
    price_change_percent
FROM suspicious_activity
ORDER BY price_change_percent DESC;
```

**Explanation:**
- Purpose: Detects tokens with sharp increases in price and transaction volume within a short period.
- Unique Insight: Helps in identifying potential market manipulation activities.

### 3.9 Analysis of Counterparty Risk Exposure

**Objective:** Assess the risk exposure of wallets based on interactions with counterparties flagged as high-risk or blacklisted.

**SQL Query:**

```sql
WITH risk_exposure AS (
    SELECT
        WALLET_ADDRESS,
        COUNTERPARTY,
        SUM(TOTAL_USD_AMOUNT) AS total_exposure,
        MAX(COUNTERPARTY_FLAG) AS counterparty_risk_flag
    FROM WALLET_COUNTERPARTY_SUMMARY
    WHERE COUNTERPARTY_FLAG = 1 -- Flag indicating high-risk counterparty
    GROUP BY WALLET_ADDRESS, COUNTERPARTY
)
SELECT
    re.WALLET_ADDRESS,
    re.COUNTERPARTY,
    re.total_exposure,
    re.counterparty_risk_flag,
    wcs.WALLET_NAME,
    wcs.WALLET_CATEGORY
FROM risk_exposure re
LEFT JOIN WALLET_COUNTERPARTY_SUMMARY wcs ON re.WALLET_ADDRESS = wcs.WALLET_ADDRESS
ORDER BY re.total_exposure DESC;
```

**Explanation:**
- Purpose: Quantifies the financial exposure wallets have to high-risk counterparties.
- Unique Insight: Essential for compliance and risk management, helping entities to mitigate potential risks.

### 3.10 Predictive Modeling for Transaction Amounts

**Objective:** Use regression analysis to predict future transaction amounts based on historical data.

**SQL Query:**

```sql
WITH historical_data AS (
    SELECT
        EXTRACT(EPOCH FROM BLOCK_TIMESTAMP)::BIGINT AS timestamp_epoch,
        USD_AMOUNT
    FROM token_transfers
    WHERE WALLET_ADDRESS = 'your_target_wallet_address'
    ORDER BY BLOCK_TIMESTAMP
),
regression_model AS (
    SELECT
        regr_slope(USD_AMOUNT, timestamp_epoch) OVER () AS slope,
        regr_intercept(USD_AMOUNT, timestamp_epoch) OVER () AS intercept
)
SELECT
    hd.timestamp_epoch,
    hd.USD_AMOUNT,
    (rm.slope * hd.timestamp_epoch + rm.intercept) AS predicted_amount
FROM historical_data hd
CROSS JOIN regression_model rm
ORDER BY hd.timestamp_epoch;
```

**Explanation:**
- Purpose: Applies linear regression to model and predict transaction amounts over time.
- Unique Insight: Provides forecasts of transaction behaviors, aiding in planning and anomaly detection.

## 4. Implementing and Extending the Queries

When implementing these queries in your environment, consider the following:

1. **Data Availability:** Ensure that any additional data (like token prices, transaction fees, or wallet entity mappings) is available in your database.

2. **Performance Optimization:** For large datasets, consider:
   - Indexing frequently queried columns
   - Optimizing your database configurations
   - Using partitioning for large tables
   - Implementing materialized views for complex, frequently-run queries

3. **Custom Thresholds:** Adjust thresholds (e.g., transaction counts, USD amounts) in the queries to suit your specific analytical needs and the characteristics of your data.

4. **Visualization:** Use the results of these queries to create visual dashboards, charts, or graphs for better interpretation and presentation. Tools like Tableau, Power BI, or custom web dashboards can be helpful.

5. **Scheduling:** Set up regular jobs to run these analyses and store results, allowing for trend analysis over time.

6. **Error Handling:** Implement proper error handling and logging to manage potential issues with data quality or query execution.

7. **Security:** Ensure that access to these queries and their results is properly secured, especially when dealing with sensitive financial data.

## 5. Potential Applications

The insights gained from these analyses can be applied in various ways:

1. **Compliance and Security:**
   - Enhance AML/KYC processes by detecting suspicious activities and high-risk interactions.
   - Identify potential money laundering schemes through circular trading detection.
   - Monitor sudden activations of dormant wallets for potential security breaches.

2. **Market Analysis:**
   - Provide clients with unique market insights, helping them make informed investment decisions.
   - Analyze cross-chain arbitrage opportunities to understand market inefficiencies.
   - Study the impact of high-frequency trading on market dynamics.

3. **Risk Management:**
   - Assist financial institutions in assessing and mitigating exposure to potential risks.
   - Quantify counterparty risk exposure for better risk assessment.
   - Predict potential market manipulations through pump and dump scheme detection.

4. **Product Development:**
   - Develop new features or services based on the insights gained from these analyses.
   - Create tools for traders to optimize their strategies based on fee consumption patterns.
   - Build predictive models for transaction amounts to aid in financial planning.

5. **Research:**
   - Contribute to academic research on blockchain economics and behavior.
   - Study long-term trends in token holding periods and their impact on price stability.

6. **Regulatory Reporting:**
   - Provide detailed reports on market activities for regulatory compliance.
   - Assist in investigations of potential market manipulations or fraudulent activities.

By leveraging these advanced SQL queries and analyses, organizations can gain deeper insights into blockchain activities, enhance their decision-making processes, and develop more sophisticated products and services in the blockchain space.
