# APY Data Miner

Daily Uniswap V3 swap transaction collector for DeFi pool performance prediction.

## Architecture

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│ EventBridge  │────▶│    Lambda    │────▶│     RDS      │
│ (00:00 UTC)  │     │  Collector   │     │  PostgreSQL  │
└──────────────┘     └──────────────┘     └──────────────┘
                           │                     │
                           ▼                     ▼
                     ┌───────────┐         ┌───────────┐
                     │  Alchemy  │         │  ML Team  │
                     │    API    │         │  (reads)  │
                     └───────────┘         └───────────┘
```

## What It Does

1. **Daily at 00:00 UTC**: EventBridge triggers Lambda
2. **Lambda Collector**: Fetches last 24h of Uniswap V3 swaps (~240K transactions)
3. **Classifies Pools**: ETH-paired, stablecoin, or other
4. **Stores in RDS**: Daily aggregated metrics per pool
5. **ML Team Queries**: Training data with rolling features

## Project Structure

```
apy-data-miner/
├── lambda/
│   └── collector/
│       ├── index.mjs           # Lambda handler
│       ├── package.json
│       └── lib/
│           ├── ethereum.mjs    # Alchemy blockchain queries
│           ├── database.mjs    # PostgreSQL operations
│           └── classifier.mjs  # Pool type classification
├── infrastructure/
│   ├── template.yaml           # AWS SAM template
│   ├── schema.sql              # PostgreSQL schema
│   └── deploy.sh               # Deployment script
└── README.md
```

## Prerequisites

- AWS CLI configured (`aws configure`)
- AWS SAM CLI installed
- Node.js 18+
- Alchemy API key (free tier works)

## Deployment

### Quick Deploy

```bash
cd infrastructure
chmod +x deploy.sh
./deploy.sh
```

### Manual Deploy

1. **Install Lambda dependencies**
   ```bash
   cd lambda/collector
   npm install
   cd ../../infrastructure
   ```

2. **Build with SAM**
   ```bash
   sam build
   ```

3. **Deploy**
   ```bash
   sam deploy --guided
   ```

   You'll be prompted for:
   - Stack name: `apy-data-miner-dev`
   - Environment: `dev` or `prod`
   - Alchemy API Key: Your key
   - DB Password: Min 8 characters

4. **Initialize database schema**

   Connect to RDS (via bastion or VPN) and run:
   ```bash
   psql -h <rds-endpoint> -U apyminer -d apyminer -f schema.sql
   ```

## Configuration

### Environment Variables

| Variable | Description | Source |
|----------|-------------|--------|
| `SECRETS_ARN` | Alchemy credentials | Secrets Manager |
| `DB_SECRETS_ARN` | Database credentials | Secrets Manager |
| `ENVIRONMENT` | dev/prod | CloudFormation |

### Local Development

Create `.env` in `lambda/collector/`:
```env
ALCHEMY_API_KEY=your_alchemy_key
DB_HOST=localhost
DB_PORT=5432
DB_NAME=apyminer
DB_USER=apyminer
DB_PASSWORD=your_password
```

Run locally:
```bash
cd lambda/collector
npm install
LOCAL_TEST=true node index.mjs
```

## Database Schema

### Tables

| Table | Purpose |
|-------|---------|
| `pools` | Pool metadata (tokens, fee tier, type) |
| `daily_metrics` | Daily tx counts per pool |
| `transactions` | Raw swap logs (optional) |
| `collection_runs` | Job execution history |

### Views for ML Team

```sql
-- Get all features for training
SELECT * FROM v_training_data;

-- Get latest features for predictions
SELECT * FROM v_pool_features
WHERE date = CURRENT_DATE - 1;
```

### Sample Query

```sql
SELECT
    pool_address,
    tx_count,
    tx_count_7d_avg,
    tx_count_7d_std,
    tx_growth_rate,
    pool_type,
    fee_tier
FROM v_pool_features
WHERE date = CURRENT_DATE - 1
ORDER BY tx_count DESC
LIMIT 100;
```

## Monitoring

### CloudWatch Logs
```
/aws/lambda/apy-data-miner-collector-{env}
```

### Manual Trigger
```bash
aws lambda invoke \
  --function-name apy-data-miner-collector-dev \
  --payload '{}' \
  response.json

cat response.json
```

### Check Last Run
```sql
SELECT * FROM collection_runs
ORDER BY created_at DESC
LIMIT 5;
```

## Costs (Estimated)

| Component | Monthly Cost |
|-----------|--------------|
| Lambda | ~$3 |
| RDS db.t3.micro | ~$15 |
| NAT Gateway | ~$35 |
| Secrets Manager | ~$1 |
| **Total (dev)** | **~$55/mo** |

To reduce costs:
- Use VPC endpoints instead of NAT Gateway (-$30)
- Use db.t3.micro with single-AZ

## Troubleshooting

### Lambda Timeout
- Check Alchemy rate limits
- Increase `BATCH_SIZE` delay
- Monitor memory usage in CloudWatch

### Database Connection Failed
- Verify Lambda is in same VPC as RDS
- Check security group allows port 5432
- Verify Secrets Manager ARN

### No Data Collected
- Check Alchemy API key is valid
- Verify block range calculation
- Look for errors in CloudWatch logs

## API for ML Team

The ML team can query RDS directly or use the provided views:

```python
import psycopg2
import pandas as pd

conn = psycopg2.connect(
    host="<rds-endpoint>",
    database="apyminer",
    user="ml_reader",
    password="<password>"
)

# Get training data
df = pd.read_sql("""
    SELECT * FROM v_training_data
    WHERE date >= CURRENT_DATE - 30
""", conn)
```

## Contributing

1. Fork the repo
2. Create feature branch
3. Test locally with `LOCAL_TEST=true`
4. Submit PR

## License

MIT
