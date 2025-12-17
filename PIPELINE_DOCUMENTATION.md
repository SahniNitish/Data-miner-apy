# APY Data Miner Pipeline - Stakeholder Documentation

**Version:** 1.0
**Last Updated:** December 17, 2025
**Owner:** Data Engineering Team

---

## Executive Summary

The APY Data Miner is an automated data collection pipeline that collects Uniswap V3 swap transaction data from the Ethereum blockchain daily. The pipeline processes, classifies, and exports feature-engineered datasets ready for machine learning model training.

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        AWS Cloud (us-east-1)                            │
│                                                                         │
│  ┌──────────────┐     ┌──────────────────┐     ┌──────────────────┐    │
│  │  EventBridge │────▶│  Collector       │────▶│  RDS PostgreSQL  │    │
│  │  (00:00 UTC) │     │  Lambda          │     │  Database        │    │
│  └──────────────┘     └──────────────────┘     └────────┬─────────┘    │
│                              │                          │               │
│                              │ Alchemy API              │               │
│                              ▼                          │               │
│                       ┌──────────────┐                  │               │
│                       │  Ethereum    │                  │               │
│                       │  Mainnet     │                  │               │
│                       └──────────────┘                  │               │
│                                                         │               │
│  ┌──────────────┐     ┌──────────────────┐             │               │
│  │  EventBridge │────▶│  Export          │◀────────────┘               │
│  │  (00:30 UTC) │     │  Lambda          │                             │
│  └──────────────┘     └────────┬─────────┘                             │
│                                │                                        │
│                                ▼                                        │
│                       ┌──────────────────┐                             │
│                       │  S3 Bucket       │                             │
│                       │  (CSV Exports)   │                             │
│                       └──────────────────┘                             │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Daily Schedule

| Time (UTC) | Time (IST) | Component | Action |
|------------|------------|-----------|--------|
| 00:00 | 05:30 | Collector Lambda | Collects 24h of Uniswap V3 swap data |
| 00:30 | 06:00 | Export Lambda | Generates feature-engineered CSVs |

---

## Data Flow

1. **Collection Phase (00:00 UTC)**
   - Fetches swap events from Uniswap V3 pools via Alchemy API
   - Classifies pools (ETH-paired, stablecoin, other)
   - Stores raw data in PostgreSQL database

2. **Export Phase (00:30 UTC)**
   - Queries database for all pool metrics
   - Calculates rolling averages, growth rates, volatility
   - Generates training/test splits (80/20)
   - Uploads CSV files to S3

---

## Data Access

### Option 1: S3 CSV Files (Recommended for ML Team)

**Bucket:** `s3://apy-data-miner-exports-226208942523/`

**Files Generated Daily:**

| File | Description | Use Case |
|------|-------------|----------|
| `pool_dataset_latest.csv` | Full feature-engineered dataset | Complete data analysis |
| `pool_training_data_latest.csv` | 80% training split | Model training |
| `pool_test_data_latest.csv` | 20% test split | Model evaluation |
| `pool_dataset_YYYY-MM-DD.csv` | Date-stamped full dataset | Historical tracking |

**Download Commands:**
```bash
# Download latest dataset
aws s3 cp s3://apy-data-miner-exports-226208942523/pool_dataset_latest.csv ./

# Download all latest files
aws s3 sync s3://apy-data-miner-exports-226208942523/ ./data/ --exclude "*" --include "*_latest.csv"

# List all available files
aws s3 ls s3://apy-data-miner-exports-226208942523/
```

### Option 2: Direct Database Access

**Connection Details:**

| Parameter | Value |
|-----------|-------|
| Host | `apy-data-miner-dev.cagkrtwngcl9.us-east-1.rds.amazonaws.com` |
| Port | `5432` |
| Database | `apyminer` |
| Username | `apyminer` |
| Password | `ApyMiner2024Secure` |
| SSL | Required |

**Connection String:**
```
postgresql://apyminer:ApyMiner2024Secure@apy-data-miner-dev.cagkrtwngcl9.us-east-1.rds.amazonaws.com:5432/apyminer?sslmode=require
```

**Python Example:**
```python
import pandas as pd
from sqlalchemy import create_engine

engine = create_engine(
    "postgresql://apyminer:ApyMiner2024Secure@"
    "apy-data-miner-dev.cagkrtwngcl9.us-east-1.rds.amazonaws.com:5432/apyminer"
)

# Query pools
pools_df = pd.read_sql("SELECT * FROM pools", engine)

# Query daily metrics
metrics_df = pd.read_sql("SELECT * FROM daily_metrics", engine)
```

**Note:** Database is in a private VPC. Access requires VPN or AWS network connectivity.

---

## Dataset Schema

### CSV Columns (25 Features)

| Column | Type | Description |
|--------|------|-------------|
| `poolAddress` | string | Uniswap V3 pool contract address |
| `date` | date | Date of the metrics |
| `tx_count` | int | Number of swap transactions |
| `unique_users` | int | Unique wallet addresses |
| `pool_name` | string | Token pair name (e.g., "WETH/USDT") |
| `token0Symbol` | string | First token symbol |
| `token1Symbol` | string | Second token symbol |
| `fee` | int | Pool fee tier in basis points |
| `fee_percentage` | float | Fee as decimal (e.g., 0.003 for 0.3%) |
| `poolType` | string | Classification: eth_paired, stablecoin, other |
| `tx_count_3d_avg` | float | 3-day rolling average transactions |
| `tx_count_7d_avg` | float | 7-day rolling average transactions |
| `tx_count_7d_std` | float | 7-day rolling standard deviation |
| `tx_count_cumulative` | int | Cumulative transactions since tracking |
| `days_since_start` | int | Days since pool first tracked |
| `day_number` | int | Sequential day number for pool |
| `tx_growth_rate` | float | Day-over-day growth rate |
| `target_tx_3d_ahead` | int | Transaction count 3 days ahead (target) |
| `target_tx_3d_avg_ahead` | float | Avg transactions next 3 days (target) |
| `target_tx_7d_ahead` | int | Transaction count 7 days ahead (target) |
| `target_tx_7d_avg_ahead` | float | Avg transactions next 7 days (target) |
| `stablecoin_pair_type` | string | stable_stable, stable_other, other |
| `activity_level` | string | high (20+), medium (5-19), low (<5) |
| `pool_maturity` | string | new (<7d), young (7-29d), mature (30-89d), established (90d+) |
| `volatility_level` | string | low_vol, medium_vol, high_vol |

### Database Tables

**pools**
```sql
CREATE TABLE pools (
    pool_address VARCHAR(42) PRIMARY KEY,
    token0 VARCHAR(42) NOT NULL,
    token1 VARCHAR(42) NOT NULL,
    token0_symbol VARCHAR(20),
    token1_symbol VARCHAR(20),
    fee_tier INTEGER NOT NULL,
    pool_type VARCHAR(20) NOT NULL,
    created_at TIMESTAMP,
    updated_at TIMESTAMP
);
```

**daily_metrics**
```sql
CREATE TABLE daily_metrics (
    id SERIAL PRIMARY KEY,
    pool_address VARCHAR(42) REFERENCES pools(pool_address),
    date DATE NOT NULL,
    tx_count INTEGER NOT NULL,
    unique_users INTEGER NOT NULL,
    created_at TIMESTAMP,
    updated_at TIMESTAMP,
    UNIQUE(pool_address, date)
);
```

---

## AWS Resources

| Resource | Name/ARN | Purpose |
|----------|----------|---------|
| Collector Lambda | `apy-data-miner-dev-CollectorFunction` | Data collection |
| Export Lambda | `apy-data-miner-query-dev` | CSV generation |
| RDS Database | `apy-data-miner-dev` | Data storage |
| S3 Bucket | `apy-data-miner-exports-226208942523` | CSV exports |
| Secrets Manager | `apy-data-miner/dev/database` | DB credentials |
| VPC | `apy-data-miner-dev-VPC` | Network isolation |

---

## Monitoring & Troubleshooting

### Check Lambda Execution Logs
```bash
# Collector logs
aws logs tail /aws/lambda/apy-data-miner-dev-CollectorFunction --follow

# Export logs
aws logs tail /aws/lambda/apy-data-miner-query-dev --follow
```

### Manual Trigger
```bash
# Run collector manually
aws lambda invoke --function-name apy-data-miner-dev-CollectorFunction result.json

# Run export manually
aws lambda invoke --function-name apy-data-miner-query-dev result.json
```

### Check Schedule Status
```bash
aws events list-rules --name-prefix apy-data-miner --region us-east-1
```

---

## Security Notes

1. **Database Access:** RDS is in a private subnet, not publicly accessible
2. **Credentials:** Stored in AWS Secrets Manager, not in code
3. **IAM:** Lambda functions use least-privilege IAM roles
4. **Encryption:** Data encrypted at rest (RDS) and in transit (SSL)

---

## Cost Estimate (Monthly)

| Service | Estimated Cost |
|---------|----------------|
| RDS (db.t3.micro) | ~$15 |
| Lambda (2 functions) | ~$1 |
| S3 Storage | ~$1 |
| Data Transfer | ~$2 |
| **Total** | **~$19/month** |

---

## Contact

For pipeline issues or access requests, contact the Data Engineering team.

---

## Appendix: Quick Reference

### AWS CLI Commands

```bash
# Download latest dataset
aws s3 cp s3://apy-data-miner-exports-226208942523/pool_dataset_latest.csv ./

# Check collector status
aws lambda get-function --function-name apy-data-miner-dev-CollectorFunction --query 'Configuration.LastModified'

# View recent logs
aws logs tail /aws/lambda/apy-data-miner-query-dev --since 1h
```

### Python Quick Start

```python
import pandas as pd
import boto3

# Load from S3
s3 = boto3.client('s3')
s3.download_file('apy-data-miner-exports-226208942523', 'pool_dataset_latest.csv', 'data.csv')
df = pd.read_csv('data.csv')

# Ready for ML
X = df[['tx_count', 'tx_count_3d_avg', 'tx_count_7d_avg', 'tx_count_7d_std',
        'tx_growth_rate', 'days_since_start', 'fee_percentage']]
y = df['target_tx_7d_avg_ahead']
```
