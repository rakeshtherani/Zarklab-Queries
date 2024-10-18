# Cryptocurrency Investment Analysis: Detailed Queries and Explanations

## Table of Contents
1. [Introduction](#introduction)
2. [Prerequisites](#prerequisites)
3. [Step-by-Step Queries with Explanations](#step-by-step-queries-with-explanations)
   3.1 [Retrieve All Transactions](#31-retrieve-all-transactions)
   3.2 [Calculate Current Holdings](#32-calculate-current-holdings)
   3.3 [Portfolio Allocation Breakdown](#33-portfolio-allocation-breakdown)
   3.4 [Portfolio Performance Over Time](#34-portfolio-performance-over-time)
   3.5 [Risk Assessment and Diversification Analysis](#35-risk-assessment-and-diversification-analysis)
   3.6 [Transaction Fee Analysis](#36-transaction-fee-analysis)
   3.7 [Realized and Unrealized Gains/Losses](#37-realized-and-unrealized-gainslosses)
   3.8 [Token Performance Comparison](#38-token-performance-comparison)
4. [Final Notes](#final-notes)

## 1. Introduction

This document provides a series of SQL queries designed to analyze a user's cryptocurrency investments. Each query is accompanied by a detailed explanation of its purpose, structure, and usage.

## 2. Prerequisites

### Tables Used:
- `token_transfers`: Contains transaction data across different chains.
- `WALLET_COUNTERPARTY_SUMMARY`: Summarizes interactions between wallets and counterparties.
- `token_prices`: Contains historical and current price data for tokens on different chains.

### User's Wallet Addresses:
The queries assume the user provides one or more wallet addresses.

## 3. Step-by-Step Queries with Explanations

### 3.1 Retrieve All Transactions

**Purpose**: Collect all transaction data related to the user's wallets to understand their activity across different blockchains.

```sql
SELECT
    tt.CHAIN,
    tt.BLOCK_TIMESTAMP,
    tt.TRANSACTION_HASH,
    tt.WALLET_ADDRESS AS user_wallet,
    tt.COUNTERPARTY AS counterparty_wallet,
    tt.DIRECTION,
    tt.TOKEN_SYMBOL,
    tt.TOKEN_AMOUNT,
    tt.USD_AMOUNT,
    tt.TRANSACTION_FEE
FROM token_transfers tt
WHERE tt.WALLET_ADDRESS IN ('user_wallet_address_1', 'user_wallet_address_2', 'user_wallet_address_3')
ORDER BY tt.BLOCK_TIMESTAMP DESC;
```

**Explanation**:
- `SELECT Clause`: Retrieves relevant columns for analysis.
- `WHERE Clause`: Filters transactions to include only those involving the user's wallets.
- `ORDER BY`: Sorts the transactions in reverse chronological order.

**Usage**: This query provides a complete list of the user's transactions, which can be used for transaction history, auditing, and further analysis.

### 3.2 Calculate Current Holdings

**Purpose**: Determine the user's current holdings for each token on each chain, considering all past transactions.

```sql
WITH user_transactions AS (
    SELECT
        tt.CHAIN,
        tt.TOKEN_SYMBOL,
        SUM(
            CASE
                WHEN tt.DIRECTION = 'received' THEN tt.TOKEN_AMOUNT
                WHEN tt.DIRECTION = 'sent' THEN -tt.TOKEN_AMOUNT
                ELSE 0
            END
        ) AS net_token_amount
    FROM token_transfers tt
    WHERE tt.WALLET_ADDRESS IN ('user_wallet_address_1', 'user_wallet_address_2', 'user_wallet_address_3')
    GROUP BY tt.CHAIN, tt.TOKEN_SYMBOL
)
SELECT
    ut.CHAIN,
    ut.TOKEN_SYMBOL,
    ut.net_token_amount,
    tp.PRICE_USD,
    (ut.net_token_amount * tp.PRICE_USD) AS total_usd_value
FROM user_transactions ut
JOIN token_prices tp ON ut.CHAIN = tp.CHAIN AND ut.TOKEN_SYMBOL = tp.TOKEN_SYMBOL
WHERE tp.TIMESTAMP = (SELECT MAX(TIMESTAMP) FROM token_prices)
ORDER BY total_usd_value DESC;
```

**Explanation**:
- `Common Table Expression (CTE) user_transactions`: Calculates the net amount of each token the user holds on each chain.
- `Main SELECT Statement`: Joins the user's net token amounts with the latest token prices to calculate current value.
- `ORDER BY`: Sorts the results by the total USD value in descending order.

**Usage**: This query gives the user a snapshot of their current holdings, including the value of each token across different chains.

### 3.3 Portfolio Allocation Breakdown

**Purpose**: Provide a detailed breakdown of the user's portfolio by token and chain, showing the percentage each holding contributes to the total portfolio value.

```sql
WITH user_holdings AS (
    SELECT
        ut.CHAIN,
        ut.TOKEN_SYMBOL,
        ut.net_token_amount,
        tp.PRICE_USD,
        (ut.net_token_amount * tp.PRICE_USD) AS total_usd_value
    FROM (
        SELECT
            tt.CHAIN,
            tt.TOKEN_SYMBOL,
            SUM(
                CASE
                    WHEN tt.DIRECTION = 'received' THEN tt.TOKEN_AMOUNT
                    WHEN tt.DIRECTION = 'sent' THEN -tt.TOKEN_AMOUNT
                    ELSE 0
                END
            ) AS net_token_amount
        FROM token_transfers tt
        WHERE tt.WALLET_ADDRESS IN ('user_wallet_address_1', 'user_wallet_address_2', 'user_wallet_address_3')
        GROUP BY tt.CHAIN, tt.TOKEN_SYMBOL
    ) ut
    JOIN token_prices tp ON ut.CHAIN = tp.CHAIN AND ut.TOKEN_SYMBOL = tp.TOKEN_SYMBOL
    WHERE tp.TIMESTAMP = (SELECT MAX(TIMESTAMP) FROM token_prices)
),
total_portfolio_value AS (
    SELECT SUM(total_usd_value) AS total_value FROM user_holdings
)
SELECT
    uh.CHAIN,
    uh.TOKEN_SYMBOL,
    uh.net_token_amount,
    uh.PRICE_USD,
    uh.total_usd_value,
    tpv.total_value,
    (uh.total_usd_value / tpv.total_value) * 100 AS allocation_percentage
FROM user_holdings uh, total_portfolio_value tpv
ORDER BY allocation_percentage DESC;
```

**Explanation**:
- `CTE user_holdings`: Calculates the user's holdings with current values.
- `CTE total_portfolio_value`: Calculates the total value of the user's portfolio.
- `Main SELECT Statement`: Calculates allocation percentage for each holding.

**Usage**: This breakdown helps the user understand how their investments are distributed, highlighting areas where they may be over- or under-invested.

### 3.4 Portfolio Performance Over Time

**Purpose**: Track the user's portfolio value over time to analyze performance and trends.

```sql
WITH daily_balances AS (
    SELECT
        DATE(tt.BLOCK_TIMESTAMP) AS date,
        tt.TOKEN_SYMBOL,
        SUM(
            CASE
                WHEN tt.DIRECTION = 'received' THEN tt.TOKEN_AMOUNT
                WHEN tt.DIRECTION = 'sent' THEN -tt.TOKEN_AMOUNT
                ELSE 0
            END
        ) OVER (PARTITION BY tt.TOKEN_SYMBOL ORDER BY DATE(tt.BLOCK_TIMESTAMP)) AS cumulative_token_amount
    FROM token_transfers tt
    WHERE tt.WALLET_ADDRESS IN ('user_wallet_address_1', 'user_wallet_address_2', 'user_wallet_address_3')
),
daily_portfolio_value AS (
    SELECT
        db.date,
        db.TOKEN_SYMBOL,
        db.cumulative_token_amount,
        tp.PRICE_USD,
        (db.cumulative_token_amount * tp.PRICE_USD) AS daily_value_usd
    FROM daily_balances db
    JOIN token_prices tp ON db.TOKEN_SYMBOL = tp.TOKEN_SYMBOL AND db.date = DATE(tp.TIMESTAMP)
)
SELECT
    dpv.date,
    SUM(dpv.daily_value_usd) AS portfolio_value_usd
FROM daily_portfolio_value dpv
GROUP BY dpv.date
ORDER BY dpv.date;
```

**Explanation**:
- `CTE daily_balances`: Calculates cumulative token amounts per day for each token.
- `CTE daily_portfolio_value`: Joins daily balances with token prices on matching dates.
- `Main SELECT Statement`: Aggregates the daily values across all tokens to get the total portfolio value per day.

**Usage**: This query provides data for plotting a portfolio value chart, showing growth or decline over time.

### 3.5 Risk Assessment and Diversification Analysis

**Purpose**: Evaluate the user's exposure to different assets and chains, identifying potential risks due to lack of diversification.

```sql
-- Use the 'user_holdings' and 'total_portfolio_value' CTEs from Step 3

SELECT
    uh.CHAIN,
    COUNT(DISTINCT uh.TOKEN_SYMBOL) AS number_of_tokens,
    SUM(uh.total_usd_value) AS total_value_per_chain,
    (SUM(uh.total_usd_value) / tpv.total_value) * 100 AS chain_allocation_percentage
FROM user_holdings uh, total_portfolio_value tpv
GROUP BY uh.CHAIN, tpv.total_value
ORDER BY chain_allocation_percentage DESC;
```

**Explanation**:
- `Columns`: Include the blockchain, number of different tokens held, total value, and allocation percentage per chain.
- `Calculations`: Sum the total value per chain and calculate the allocation percentage.

**Usage**: Helps the user understand their exposure to different blockchains, identifying overconcentration in a single chain, which may pose a risk.

### 3.6 Transaction Fee Analysis

**Purpose**: Analyze the total fees the user has paid across all transactions, helping them understand costs associated with their activities.

```sql
SELECT
    tt.CHAIN,
    SUM(tt.TRANSACTION_FEE) AS total_fees_paid,
    COUNT(*) AS number_of_transactions,
    (SUM(tt.TRANSACTION_FEE) / COUNT(*)) AS average_fee_per_transaction
FROM token_transfers tt
WHERE tt.WALLET_ADDRESS IN ('user_wallet_address_1', 'user_wallet_address_2', 'user_wallet_address_3')
GROUP BY tt.CHAIN
ORDER BY total_fees_paid DESC;
```

**Explanation**:
- `Columns`: Include the blockchain, total fees paid, number of transactions, and average fee per transaction.
- `Calculations`: Sum total fees, count transactions, and calculate average fee.

**Usage**: This analysis helps the user see where they are incurring the most fees, potentially informing decisions to adjust transaction behavior or switch to chains with lower fees.

### 3.7 Realized and Unrealized Gains/Losses

**Purpose**: Calculate the user's realized gains or losses from completed transactions and unrealized gains or losses from current holdings.

```sql
-- Query for Realized Gains/Losses
WITH token_purchases AS (
    SELECT
        tt.TOKEN_SYMBOL,
        SUM(
            CASE
                WHEN tt.DIRECTION = 'received' THEN tt.TOKEN_AMOUNT
                ELSE 0
            END
        ) AS total_purchased,
        SUM(
            CASE
                WHEN tt.DIRECTION = 'received' THEN tt.USD_AMOUNT
                ELSE 0
            END
        ) AS total_purchase_cost
    FROM token_transfers tt
    WHERE tt.WALLET_ADDRESS IN ('user_wallet_address_1', 'user_wallet_address_2', 'user_wallet_address_3')
    GROUP BY tt.TOKEN_SYMBOL
),
token_sales AS (
    SELECT
        tt.TOKEN_SYMBOL,
        SUM(
            CASE
                WHEN tt.DIRECTION = 'sent' THEN tt.TOKEN_AMOUNT
                ELSE 0
            END
        ) AS total_sold,
        SUM(
            CASE
                WHEN tt.DIRECTION = 'sent' THEN tt.USD_AMOUNT
                ELSE 0
            END
        ) AS total_sale_proceeds
    FROM token_transfers tt
    WHERE tt.WALLET_ADDRESS IN ('user_wallet_address_1', 'user_wallet_address_2', 'user_wallet_address_3')
    GROUP BY tt.TOKEN_SYMBOL
)
SELECT
    tp.TOKEN_SYMBOL,
    tp.total_purchased,
    tp.total_purchase_cost,
    ts.total_sold,
    ts.total_sale_proceeds,
    (ts.total_sale_proceeds - (tp.total_purchase_cost * (ts.total_sold / tp.total_purchased))) AS realized_gain_loss
FROM token_purchases tp
JOIN token_sales ts ON tp.TOKEN_SYMBOL = ts.TOKEN_SYMBOL;

-- Query for Unrealized Gains/Losses
SELECT
    uh.TOKEN_SYMBOL,
    uh.net_token_amount AS current_holdings,
    uh.total_usd_value AS current_value_usd,
    (tp.total_purchase_cost - ts.total_sale_proceeds) AS remaining_cost_basis,
    (uh.total_usd_value - (tp.total_purchase_cost - ts.total_sale_proceeds)) AS unrealized_gain_loss
FROM user_holdings uh
JOIN token_purchases tp ON uh.TOKEN_SYMBOL = tp.TOKEN_SYMBOL
JOIN token_sales ts ON uh.TOKEN_SYMBOL = ts.TOKEN_SYMBOL;
```

**Explanation**:
- `Realized Gains/Losses`: Calculates profits or losses from completed transactions.
- `Unrealized Gains/Losses`: Estimates potential profits or losses based on current market prices.

**Usage**: Helps the user understand their profit or loss from trading activities and the potential profit or loss if they were to sell their current holdings at current market prices.

### 3.8 Token Performance Comparison

**Purpose**: Compare the user's tokens' performance against market benchmarks or other tokens.

```sql
WITH token_returns AS (
    SELECT
        tp.TOKEN_SYMBOL,
        ((tp.PRICE_USD / tp_start.PRICE_USD) - 1) * 100 AS percentage_return
    FROM token_prices tp
    JOIN (
        SELECT TOKEN_SYMBOL, MIN(TIMESTAMP) AS start_date
        FROM token_prices
        GROUP BY TOKEN_SYMBOL
    ) tp_start ON tp.TOKEN_SYMBOL = tp_start.TOKEN_SYMBOL
    WHERE tp.TIMESTAMP = (SELECT MAX(TIMESTAMP) FROM token_prices)
)
SELECT
    tr.TOKEN_SYMBOL,
    tr.percentage_return,
    CASE
        WHEN tr.percentage_return > benchmark_return THEN 'Outperformed'
        ELSE 'Underperformed'
    END AS performance_vs_benchmark
FROM token_returns tr
CROSS JOIN (
    SELECT ((benchmark_current.PRICE_USD / benchmark_start.PRICE_USD) - 1) * 100 AS benchmark_return
    FROM (
        SELECT PRICE_USD FROM token_prices WHERE TOKEN_SYMBOL = 'BENCHMARK_TOKEN' AND TIMESTAMP = (SELECT MAX(TIMESTAMP) FROM token_prices)
    ) benchmark_current
    CROSS JOIN (
        SELECT PRICE_USD FROM token_prices WHERE TOKEN_SYMBOL = 'BENCHMARK_TOKEN' AND TIMESTAMP = (SELECT MIN(TIMESTAMP) FROM token_prices WHERE TOKEN_SYMBOL = 'BENCHMARK_TOKEN')
    ) benchmark_start
) benchmark
ORDER BY tr.percentage_return DESC;
```

**Explanation**:
- `CTE token_returns`: Calculates the percentage return of each token since the earliest available price.
- `Benchmark Return Calculation`: Computes the percentage return of a benchmark token (e.g., BTC or ETH).
- `Main SELECT Statement`: Compares each token's return to the benchmark and indicates whether the token outperformed or underperformed the benchmark.

**Usage**: Helps the user assess the performance of their investments relative to the broader market or a specific benchmark token.

## 4. Final Notes

### Data Accuracy
- Ensure that all data used is accurate and up-to-date, especially price data for valuations and gains/losses calculations.
- Regularly update and validate data sources to maintain the integrity of the analysis.

### Data Privacy
- Handle user data securely and in compliance with privacy regulations.
- Implement appropriate encryption and access control measures to protect sensitive financial information.

### User Interface
- Present the results in a user-friendly format, such as dashboards, charts, and tables.
- Consider using interactive visualizations to allow users to explore their data more deeply.

### Limitations
1. **Historical Prices**: 
   - Accurate gains/losses calculations require precise historical price data at the times of transactions.
   - Ensure that the `token_prices` table has comprehensive historical data for all tokens and chains.

2. **Tax Implications**: 
   - The calculations provided are for informational purposes and may not reflect exact tax liabilities.
   - Advise users to consult with tax professionals for accurate tax calculations based on their specific situations.

3. **Chain-Specific Considerations**:
   - Different blockchain networks may have unique characteristics that affect transaction processing, fees, or token behavior.
   - Ensure that the analysis accounts for chain-specific nuances where relevant.

4. **Data Completeness**:
   - The accuracy of the analysis depends on having a complete record of all user transactions across all relevant wallets and chains.
   - Implement mechanisms to detect and flag potential data gaps or inconsistencies.

### Performance Optimization
- For users with extensive transaction histories, consider implementing query optimization techniques or data pre-aggregation to improve response times.
- Use appropriate indexing on frequently queried columns to enhance query performance.

### Customization and Scalability
- Design the system to allow for easy addition of new analytical queries as user needs evolve.
- Consider implementing parameterized queries to allow for customization of analysis based on user preferences (e.g., date ranges, specific tokens of interest).

### Educational Resources
- Provide explanations and educational content alongside the analysis to help users interpret the results effectively.
- Consider offering tooltips, glossaries, or links to more detailed resources for complex financial concepts.

### Regulatory Compliance
- Stay informed about and comply with relevant financial regulations and reporting requirements in the jurisdictions where the service is offered.
- Implement necessary disclaimers and user agreements to clarify the nature and limitations of the provided analysis.

By providing these detailed queries with explanations, you offer users a comprehensive understanding of their cryptocurrency portfolios. This level of detail enhances the value of your analysis and demonstrates transparency in how the insights are derived. Remember to continuously refine and expand these analyses based on user feedback and evolving market conditions to ensure the tool remains valuable and relevant to cryptocurrency investors.

