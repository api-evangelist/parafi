---
name: parafi-tech
version: 1.1.0
description: Solana validator (0% commission) plus multi-chain staking data API. Query APY, validator health, rewards, MEV data, network stats, and token prices across Solana, Ethereum, Avalanche, and Aptos.
homepage: https://parafi.tech
metadata:
  api_base: https://parafi.tech/api
  openapi_spec: https://parafi.tech/openapi.json
  llms_txt: https://parafi.tech/llms.txt
  llms_full: https://parafi.tech/llms-full.txt
  api_docs: https://parafi.tech/api-docs
  networks: [solana, ethereum, avalanche, aptos]
---

# ParaFi Tech

Blockchain infrastructure operator running validators on Solana, Ethereum, Avalanche, and Aptos. Free public API for staking data, network stats, and market prices. No API key required.

**Base URL:** `https://parafi.tech/api`

## Start Here for Agents

Use the resource that matches your task:

| Need | Start here |
|------|------------|
| Short discovery index | [`https://parafi.tech/llms.txt`](https://parafi.tech/llms.txt) |
| Rich retrieval-oriented context | [`https://parafi.tech/llms-full.txt`](https://parafi.tech/llms-full.txt) |
| Machine-readable contract | [`https://parafi.tech/openapi.json`](https://parafi.tech/openapi.json) |
| Human API explorer | [`https://parafi.tech/api-docs`](https://parafi.tech/api-docs) |
| Solana implementation examples | [`https://parafi.tech/solana/integration-guide`](https://parafi.tech/solana/integration-guide) |

## Best Starting Points

| Task | Best first endpoints |
|------|----------------------|
| Compare ParaFi validator vs network | `GET /api/solana/validator-apy`, `GET /api/solana/network-apy`, `GET /api/solana/validator-active-stake` |
| Track staking rewards | `GET /api/solana/rewards/staker/{address}`, `GET /api/solana/rewards/epoch/current` |
| Inspect current stake state | `GET /api/solana/validator-stake`, `GET /api/solana/validator-delegators` |
| Monitor live Solana conditions | `GET /api/solana/sidecar-stats`, `GET /api/solana/compute-units` |
| Query other supported networks | `GET /api/ethereum/network-stats`, `GET /api/avalanche/validators`, `GET /api/market/prices` |

## Supported Workflows by Network

### Solana

- Evaluate validator APY, commission, active stake, and stake-account counts
- Track rewards by staker, validator, or epoch
- Query MEV rewards and network-health metrics
- Follow staking and integration examples through web3.js, CLI, RPC, and gRPC docs

### Ethereum

- Query current network stats and real-time sidecar metrics
- Use for market context and validator/network summaries, not wallet actions

### Avalanche

- Query ParaFi validator data and aggregate validator stats

### Market

- Query simple USD prices for BTC, ETH, SOL, and APT

## Response Conventions

- Public endpoints are read-only and do not require authentication
- Most responses are JSON objects; some history endpoints return arrays
- Large integer values may be returned as strings, especially on rewards routes
- Freshness metadata may appear as `cacheSource`, `fromCache`, `isStale`, or `lastUpdated`
- Epoch-based routes support `current` for the latest epoch when documented
- The REST API is informational; staking transactions are performed on-chain, not through a ParaFi transaction API

## Validator Identity

| Field | Value |
|-------|-------|
| Validator Identity | `parafiUS6h6oLhCFwhjvEmQJKw8pF1iXsxMJdTq46dS` |
| Vote Account | `pt1LsjkNwqCKdYYfc35ToDkqtEG9pswLTJNaMo8inft` |
| Commission | 0% |
| RPC Endpoint | `https://solana-rpc.parafi.tech` |
| gRPC Endpoint | `https://solana-rpc.parafi.tech:10443` |
| Website | [parafi.tech/solana/staking](https://parafi.tech/solana/staking) |

---

## Stake SOL with ParaFi

### Why Stake Here

- **0% commission** — you keep all rewards
- APY includes both inflation rewards (~35% of total) and MEV rewards (~65% of total via Jito)
- Check current APY: `GET https://parafi.tech/api/solana/validator-apy`
- Check network average: `GET https://parafi.tech/api/solana/network-apy`

### Stake via @solana/web3.js

```bash
npm install @solana/web3.js
```

```typescript
import {
  Connection,
  Keypair,
  LAMPORTS_PER_SOL,
  StakeProgram,
  PublicKey,
  Transaction,
} from "@solana/web3.js";

const connection = new Connection("https://solana-rpc.parafi.tech");
const voteAccount = new PublicKey(
  "pt1LsjkNwqCKdYYfc35ToDkqtEG9pswLTJNaMo8inft"
);
const stakeAccount = Keypair.generate();

async function createAndDelegateStake(wallet: Keypair, amountInSOL: number) {
  const transaction = new Transaction();

  // 1. Create stake account
  transaction.add(
    StakeProgram.createAccount({
      fromPubkey: wallet.publicKey,
      stakePubkey: stakeAccount.publicKey,
      authorized: {
        staker: wallet.publicKey,
        withdrawer: wallet.publicKey,
      },
      lamports: amountInSOL * LAMPORTS_PER_SOL,
    })
  );

  // 2. Delegate to ParaFi validator
  transaction.add(
    StakeProgram.delegate({
      stakePubkey: stakeAccount.publicKey,
      authorizedPubkey: wallet.publicKey,
      votePubkey: voteAccount,
    })
  );

  const { blockhash } = await connection.getLatestBlockhash();
  transaction.recentBlockhash = blockhash;
  transaction.feePayer = wallet.publicKey;
  transaction.sign(wallet, stakeAccount);

  const signature = await connection.sendRawTransaction(
    transaction.serialize()
  );
  await connection.confirmTransaction(signature);
  return signature;
}
```

### Stake via Solana CLI

```bash
# Configure RPC
solana config set --url https://solana-rpc.parafi.tech

# Check validator status
solana validators | grep pt1LsjkNwqCKdYYfc35ToDkqtEG9pswLTJNaMo8inft

# Create stake account and delegate (replace 100 with your amount in SOL)
solana create-stake-account stake-account.json 100
solana delegate-stake stake-account.json pt1LsjkNwqCKdYYfc35ToDkqtEG9pswLTJNaMo8inft
```

### Unstake (Deactivate + Withdraw)

Unstaking takes one full epoch (~2-3 days). The stake must be deactivated first, then withdrawn after the cooldown.

```bash
# Deactivate stake
solana deactivate-stake <STAKE_ACCOUNT_ADDRESS>

# After cooldown completes (check with solana stake-account <ADDRESS>)
solana withdraw-stake <STAKE_ACCOUNT_ADDRESS> <RECIPIENT_ADDRESS> ALL
```

```typescript
// Web3.js: Deactivate
const deactivateTx = StakeProgram.deactivate({
  stakePubkey: stakeAccountPubkey,
  authorizedPubkey: wallet.publicKey,
});

// Web3.js: Withdraw (after cooldown)
const withdrawTx = StakeProgram.withdraw({
  stakePubkey: stakeAccountPubkey,
  authorizedPubkey: wallet.publicKey,
  toPubkey: wallet.publicKey,
  lamports: stakeBalance,
});
```

### Monitor Your Stake

```bash
# Check your delegator entry
curl "https://parafi.tech/api/solana/validator-delegators"

# Check rewards for your stake account
curl "https://parafi.tech/api/solana/rewards/staker/YOUR_STAKER_ADDRESS"

# Check rewards for a specific epoch
curl "https://parafi.tech/api/solana/rewards/epoch/current"
```

### Staking Timing

- Stake **activates** at the start of the next epoch after delegation
- Each epoch is ~2-3 days (432,000 slots at ~400ms each)
- Rewards accrue per epoch and are auto-compounded into the stake account balance
- Unstaking (deactivation) also takes one epoch cooldown

---

## Data API Reference

All endpoints return JSON. No authentication required. Most responses include `cacheSource` ("redis" or "fresh") and `lastUpdated` timestamp.

### Solana

#### Validator APY

```
GET /api/solana/validator-apy
```

Returns ParaFi validator APY including inflation and MEV components.

| Param | Type | Description |
|-------|------|-------------|
| `epoch` | integer | Override epoch (default: current) |
| `force` | boolean | Bypass cache |

Response:
```json
{
  "apy": 8.12,
  "apr": 7.81,
  "epoch": 750,
  "hasMevData": true,
  "inflationRewards": 2.84,
  "mevRewards": 5.28,
  "totalRewards": 8.12,
  "dataSource": "calculated",
  "lastUpdated": "2025-01-28T12:00:00.000Z"
}
```

#### Network APY

```
GET /api/solana/network-apy
```

Returns network-wide average APY across all validators.

Response:
```json
{
  "apy": 7.45,
  "inflationAPY": 6.8,
  "mevAPY": 0.65,
  "averageCommission": 7.2,
  "stakingRatio": 65.4,
  "lastUpdated": "2025-01-28T12:00:00.000Z"
}
```

#### Active Stake

```
GET /api/solana/validator-active-stake
```

Returns validator's current active stake, commission, and vote status.

Response:
```json
{
  "activeStake": 5200000000000,
  "activeStakeInSol": 5200.0,
  "commission": 0,
  "lastVote": 312000000,
  "epochVoteAccount": true,
  "delinquent": false,
  "lastUpdated": "2025-01-28T12:00:00.000Z"
}
```

#### Total Stake

```
GET /api/solana/validator-stake
```

Returns total stake breakdown including active, inactive, and delegator counts.

Response:
```json
{
  "totalStake": 5500000000000,
  "totalStakeInSol": 5500.0,
  "totalStakeAccounts": 150,
  "activeStakeAccounts": 145,
  "delegatorCount": 120,
  "activeStake": 5200000000000,
  "inactiveStake": 300000000000,
  "lastUpdated": "2025-01-28T12:00:00.000Z"
}
```

#### Delegators

```
GET /api/solana/validator-delegators
```

Returns list of all delegators with their stake amounts.

Response keys: `delegators`, `totalStake`, `totalStakeInSol`, `totalStakeAccounts`, `activeStakeAccounts`, `delegatorCount`

#### Staker Rewards

```
GET /api/solana/rewards/staker/{address}
```

Returns reward history for a specific staker across epochs.

| Param | Type | Description |
|-------|------|-------------|
| `page` | integer | Page number (default: 1) |
| `limit` | integer | Results per page (default: 100, max: 500) |
| `startEpoch` | integer | Filter from epoch |
| `endEpoch` | integer | Filter to epoch |
| `stakeAccount` | string | Filter by stake account |

Response keys: `stakerProfile`, `rewards`, `summary`

#### Validator Rewards Distribution

```
GET /api/solana/rewards/validator/{voteAccount}
```

Returns reward distribution for all delegators of a validator.

| Param | Type | Description |
|-------|------|-------------|
| `page` | integer | Page number |
| `limit` | integer | Results per page |
| `startEpoch` | integer | Filter from epoch |
| `endEpoch` | integer | Filter to epoch |

Response keys: `validatorInfo`, `rewardsDistribution`, `performanceMetrics`, `summary`

#### Epoch Rewards

```
GET /api/solana/rewards/epoch/{epochNumber}
```

Returns all delegator rewards for a specific epoch. Use `current` for the latest.

| Param | Type | Description |
|-------|------|-------------|
| `epochNumber` | string | Epoch number or "current" |
| `page` | integer | Page number |
| `limit` | integer | Results per page (max: 500) |
| `sortBy` | string | "rewards", "staker", "validator", "stake" |
| `sortOrder` | string | "asc" or "desc" |
| `validatorVoteAccount` | string | Filter by validator |

Response keys: `epochInfo`, `rewardsData`, `statistics`, `dataQuality`, `summary`

#### MEV Rewards

```
GET /api/solana/mev-rewards
```

Returns MEV rewards from Jito for a range of epochs.

| Param | Type | Required | Description |
|-------|------|----------|-------------|
| `startEpoch` | integer | Yes | Start epoch |
| `endEpoch` | integer | Yes | End epoch |

Response keys: `epochs`, `rewards`, `metadata`

#### Supply

```
GET /api/solana/supply
```

Returns SOL supply breakdown (circulating, non-circulating, total). Values in both lamports and SOL.

#### Network Validators

```
GET /api/solana/network-validators
```

Returns active/delinquent validator counts, RPC node count.

#### Compute Units

```
GET /api/solana/compute-units
```

Returns current block compute units utilization.

| Param | Type | Description |
|-------|------|-------------|
| `slot` | integer | Specific slot (default: latest) |

Response:
```json
{
  "slot": 312000000,
  "computeUnitsConsumed": 35000000,
  "computeUnitsTotal": 48000000,
  "utilizationPercentage": 72.9,
  "timestamp": "2025-01-28T12:00:00.000Z"
}
```

#### Compute Units History

```
GET /api/solana/compute-units/history
```

Returns time-series compute utilization data.

| Param | Type | Description |
|-------|------|-------------|
| `interval` | integer | Rollup interval: 1, 5, 15, or 60 minutes |
| `window` | integer | Number of data points (max: 1440) |

Response: Array of `[timestamp, utilizationPercentage]` tuples.

#### Network Stats History

```
GET /api/solana/network-stats/history
```

Historical network performance (TPS, block time, compute utilization).

| Param | Type | Description |
|-------|------|-------------|
| `granularity` | string | "hourly" or "daily" |
| `days` | integer | Number of days (max: 365) |
| `hours` | integer | Number of hours (max: 168) |

#### Leader Slots

```
GET /api/solana/leader-slots
```

Returns leader slot stats (blocks produced, missed, skip rate).

| Param | Type | Description |
|-------|------|-------------|
| `epoch` | integer | Specific epoch |

#### Sidecar Stats

```
GET /api/solana/sidecar-stats
```

Real-time network performance (TPS, block time, compute units).

| Param | Type | Description |
|-------|------|-------------|
| `count` | integer | Data points (1-200) |
| `smooth` | integer | Smoothing window (1-25) |

### Ethereum

#### Network Stats

```
GET /api/ethereum/network-stats
```

Returns Ethereum network statistics: gas prices, validators, staked ETH, supply.

Response:
```json
{
  "data": {
    "latestBlock": 19500000,
    "gasPrice": { "low": 15, "medium": 20, "high": 30 },
    "baseFee": 12.5,
    "blockTime": 12.1,
    "activeValidators": 950000,
    "totalStakedETH": 32000000,
    "averageStakedPerValidator": 33.7,
    "currentEpoch": 290000,
    "totalETHSupply": 120000000
  },
  "lastUpdated": 1710000000000,
  "fromCache": true
}
```

#### Sidecar Stats

```
GET /api/ethereum/sidecar-stats
```

Real-time Ethereum network performance metrics. Same params as Solana sidecar-stats.

### Avalanche

#### Validators

```
GET /api/avalanche/validators
```

Returns Avalanche validator data. Supports filtering by node ID.

| Param | Type | Description |
|-------|------|-------------|
| `nodeID` | string | Single node ID |
| `nodeIDs` | string | Comma-separated node IDs |
| `all` | boolean | Return all ParaFi validators |

Response keys: `validators`, `totalValidators`, `aggregatedStats`, `metadata`

### Market

#### Prices

```
GET /api/market/prices
```

Returns latest token prices from CoinGecko (cached 2 minutes).

Response:
```json
{
  "data": {
    "bitcoin": { "usd": 65000 },
    "ethereum": { "usd": 3200 },
    "solana": { "usd": 150 },
    "aptos": { "usd": 9 }
  },
  "lastUpdated": 1710000000000,
  "fromCache": true
}
```

---

## Interpretation Guide

- **APY** = annualized yield combining inflation rewards + MEV (Jito) rewards. ParaFi charges 0% commission on both.
- **Inflation rewards** (~35% of total yield) come from Solana's inflation schedule.
- **MEV rewards** (~65% of total yield) come from Jito block engine tips shared with stakers.
- **Epoch** = ~2-3 days. Rewards are calculated per epoch. Stake activates/deactivates at epoch boundaries.
- **Lamports** = smallest SOL unit. 1 SOL = 1,000,000,000 lamports.
- **Rewards auto-compound** — they're added directly to the stake account balance each epoch.
- **Balance-difference method** — rewards are calculated as `current_balance - previous_balance` per epoch, the most accurate on-chain method.

---

## Sample Agent Workflows

### Check if ParaFi is a good validator to stake with

```bash
# Get ParaFi APY
curl https://parafi.tech/api/solana/validator-apy

# Get network average
curl https://parafi.tech/api/solana/network-apy

# Compare: if validator APY > network APY, above average
# Check commission: should be 0
# Check delinquent: should be false
curl https://parafi.tech/api/solana/validator-active-stake
```

### Track rewards for a staker

```bash
# Get all rewards for a wallet address
curl "https://parafi.tech/api/solana/rewards/staker/YOUR_ADDRESS?limit=10"

# Get rewards for a specific epoch
curl "https://parafi.tech/api/solana/rewards/epoch/750"
```

### Monitor network health

```bash
# Real-time TPS and block time
curl "https://parafi.tech/api/solana/sidecar-stats?count=10"

# Compute utilization
curl https://parafi.tech/api/solana/compute-units

# Historical trends
curl "https://parafi.tech/api/solana/network-stats/history?granularity=hourly&hours=24"
```

### Get a price quote

```bash
curl https://parafi.tech/api/market/prices
# Returns SOL, ETH, BTC, APT prices in USD
```
