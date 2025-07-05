# EulerSwap Subgraph Development Plan

## Project Overview

Create a comprehensive subgraph for EulerSwap to provide real market data for delta-neutral LP strategy backtesting. This replaces the synthetic data currently used in the `euler-delta-neutral` project with actual on-chain transaction data.

## Context & Background

### Parent Project
This subgraph supports a **delta-neutral LP backtesting strategy** project located at `../euler-delta-neutral/`. The parent project currently uses synthetic data generation but needs real EulerSwap data for:

- Historical strategy backtesting (since June 2024)
- Real-time delta calculation and rebalancing
- Actual vault borrowing costs and liquidity constraints
- Educational demonstration of delta-neutral concepts

### EulerSwap Protocol
EulerSwap is an innovative AMM that provides **40x liquidity depth** by integrating with Euler's lending vaults for just-in-time liquidity. See `research-context.md` for complete technical background.

## Target Network: Ethereum Mainnet

**Primary Focus**: Ethereum Mainnet (Chain ID: 1)
- **Factory**: `0xb013be1D0D380C13B58e889f412895970A2Cf228`
- **Implementation**: `0xc35a0FDA69e9D71e68C0d9CBb541Adfd21D6B117`
- **Periphery**: `0x208fF5Eb543814789321DaA1B5Eb551881D6B06`
- **Deployment**: June 2024 (start indexing from this block)
- **Live Testing**: $500k CTF deployed for real production testing

## Phase 1: Project Setup (30 minutes)

### 1.1 Environment Setup
```bash
# Install The Graph CLI
npm install -g @graphprotocol/graph-cli

# Initialize new subgraph project
graph init --product hosted-service eulerswap-mainnet

# Install dependencies
npm install
```

### 1.2 Project Configuration
- Configure for Ethereum mainnet
- Set start block to June 2024 EulerSwap deployment
- Add provided contract addresses from `deployment-addresses.json`

## Phase 2: Contract Integration (1 hour)

### 2.1 ABI Integration
Use provided interfaces in `contracts/interfaces/`:
- `IEulerSwap.sol` - Core pool contract
- `IEulerSwapFactory.sol` - Factory for pool deployment
- `IEulerSwapPeriphery.sol` - User-facing helper functions

### 2.2 Event Identification
Key events to track (infer from contract functions):

**Factory Events**:
```solidity
event PoolDeployed(address indexed pool, address indexed asset0, address indexed asset1, ...);
event PoolUninstalled(address indexed pool);
```

**Pool Events**:
```solidity
event Swap(address indexed sender, uint amount0In, uint amount1In, uint amount0Out, uint amount1Out, address indexed to);
event Sync(uint112 reserve0, uint112 reserve1);
```

**Integration Events** (from Euler vaults):
- Borrow/Repay events
- Collateral state changes
- Utilization updates

### 2.3 Contract Configuration
```yaml
# subgraph.yaml
dataSources:
  - kind: ethereum/contract
    name: EulerSwapFactory
    network: mainnet
    source:
      address: "0xb013be1D0D380C13B58e889f412895970A2Cf228"
      abi: EulerSwapFactory
      startBlock: [JUNE_2024_BLOCK]
```

## Phase 3: GraphQL Schema Design (45 minutes)

### 3.1 Core Entities

```graphql
type Pool @entity {
  id: ID!                          # Pool address
  asset0: Token!                   # First token
  asset1: Token!                   # Second token
  vault0: Bytes!                   # Euler vault address
  vault1: Bytes!                   # Euler vault address
  eulerAccount: Bytes!             # Associated Euler account
  factory: Factory!                # Factory that created this pool
  
  # Pool configuration
  equilibriumReserve0: BigInt!
  equilibriumReserve1: BigInt!
  fee: BigInt!
  protocolFee: BigInt!
  
  # Current state
  reserve0: BigInt!
  reserve1: BigInt!
  status: Int!                     # 0=unactivated, 1=unlocked, 2=locked
  
  # Derived fields for delta-neutral strategies
  totalVolumeUSD: BigDecimal!
  liquidityUSD: BigDecimal!
  
  # Relationships
  swaps: [Swap!]! @derivedFrom(field: "pool")
  
  # Timestamps
  createdAtTimestamp: BigInt!
  createdAtBlockNumber: BigInt!
}

type Swap @entity {
  id: ID!                          # Transaction hash + log index
  pool: Pool!                      # Pool where swap occurred
  transaction: Transaction!         # Transaction details
  
  # Swap details
  sender: Bytes!
  recipient: Bytes!
  amount0In: BigInt!
  amount1In: BigInt!
  amount0Out: BigInt!
  amount1Out: BigInt!
  
  # Derived calculations
  amountUSD: BigDecimal!
  priceImpact: BigDecimal!
  
  # Delta-neutral specific fields
  deltaExposure: BigDecimal!       # LP position delta after swap
  hedgeRequired: BigDecimal!       # Required hedge amount
  
  # Timestamps
  timestamp: BigInt!
  blockNumber: BigInt!
}

type VaultState @entity {
  id: ID!                          # vault address + block number
  vault: Bytes!
  pool: Pool!
  
  # Vault metrics
  totalSupply: BigInt!
  totalBorrow: BigInt!
  utilizationRate: BigDecimal!
  supplyAPY: BigDecimal!
  borrowAPY: BigDecimal!
  availableLiquidity: BigInt!
  
  # Derived fields
  liquidityUSD: BigDecimal!
  
  # Timestamps
  timestamp: BigInt!
  blockNumber: BigInt!
}

type DeltaNeutralMetrics @entity {
  id: ID!                          # pool address + timestamp
  pool: Pool!
  
  # Delta calculations
  lpDelta: BigDecimal!             # Current LP position delta
  hedgeRatio: BigDecimal!          # Required hedge ratio
  impermanentLoss: BigDecimal!     # Current IL percentage
  
  # Strategy metrics
  totalValueLocked: BigDecimal!
  borrowingCost: BigDecimal!
  tradingFeeAPY: BigDecimal!
  
  # Timestamps
  timestamp: BigInt!
  blockNumber: BigInt!
}
```

### 3.2 Supporting Entities

```graphql
type Factory @entity {
  id: ID!                          # Factory address
  poolCount: BigInt!
  totalVolumeUSD: BigDecimal!
  
  pools: [Pool!]! @derivedFrom(field: "factory")
}

type Token @entity {
  id: ID!                          # Token address
  symbol: String!
  name: String!
  decimals: BigInt!
  
  # Price tracking
  derivedETH: BigDecimal!
  
  # Usage in pools
  pools0: [Pool!]! @derivedFrom(field: "asset0")
  pools1: [Pool!]! @derivedFrom(field: "asset1")
}

type Transaction @entity {
  id: ID!                          # Transaction hash
  blockNumber: BigInt!
  timestamp: BigInt!
  gasUsed: BigInt!
  gasPrice: BigInt!
  
  swaps: [Swap!]! @derivedFrom(field: "transaction")
}
```

## Phase 4: Event Handlers Implementation (2 hours)

### 4.1 Factory Event Handlers

```typescript
// src/mappings/factory.ts
export function handlePoolDeployed(event: PoolDeployedEvent): void {
  let pool = new Pool(event.params.pool.toHex())
  let factory = loadOrCreateFactory(event.address)
  
  // Set pool parameters from event
  pool.factory = factory.id
  pool.asset0 = event.params.asset0
  pool.asset1 = event.params.asset1
  pool.vault0 = event.params.vault0
  pool.vault1 = event.params.vault1
  
  // Initialize derived fields
  pool.totalVolumeUSD = ZERO_BD
  pool.liquidityUSD = ZERO_BD
  
  pool.save()
  
  // Create dynamic data source for pool events
  PoolTemplate.create(event.params.pool)
}
```

### 4.2 Pool Event Handlers

```typescript
// src/mappings/pool.ts
export function handleSwap(event: SwapEvent): void {
  let pool = Pool.load(event.address.toHex())!
  let transaction = loadOrCreateTransaction(event.transaction.hash)
  
  let swap = new Swap(
    event.transaction.hash.toHex() + "-" + event.logIndex.toString()
  )
  
  swap.pool = pool.id
  swap.transaction = transaction.id
  swap.sender = event.params.sender
  swap.recipient = event.params.to
  swap.amount0In = event.params.amount0In
  swap.amount1In = event.params.amount1In
  swap.amount0Out = event.params.amount0Out
  swap.amount1Out = event.params.amount1Out
  
  // Calculate USD value and price impact
  swap.amountUSD = calculateSwapValueUSD(swap)
  swap.priceImpact = calculatePriceImpact(pool, swap)
  
  // Delta-neutral specific calculations
  swap.deltaExposure = calculateLPDelta(pool, swap)
  swap.hedgeRequired = calculateHedgeAmount(swap.deltaExposure)
  
  swap.timestamp = event.block.timestamp
  swap.blockNumber = event.block.number
  
  swap.save()
  
  // Update pool state
  updatePoolState(pool, event)
  
  // Update delta-neutral metrics
  updateDeltaNeutralMetrics(pool, event.block)
}
```

### 4.3 Vault Integration Handlers

```typescript
// src/mappings/vault.ts
export function handleVaultStateChange(event: VaultEvent): void {
  let vaultStateId = event.address.toHex() + "-" + event.block.number.toString()
  let vaultState = new VaultState(vaultStateId)
  
  vaultState.vault = event.address
  vaultState.totalSupply = getVaultTotalSupply(event.address)
  vaultState.totalBorrow = getVaultTotalBorrow(event.address)
  vaultState.utilizationRate = calculateUtilizationRate(vaultState)
  
  // Calculate APYs from vault contracts
  vaultState.supplyAPY = getVaultSupplyAPY(event.address)
  vaultState.borrowAPY = getVaultBorrowAPY(event.address)
  
  vaultState.timestamp = event.block.timestamp
  vaultState.blockNumber = event.block.number
  
  vaultState.save()
}
```

## Phase 5: Delta-Neutral Calculations (1 hour)

### 5.1 Core Delta Functions

```typescript
// src/utils/delta.ts
import { BigDecimal, BigInt } from "@graphprotocol/graph-ts"

export function calculateLPDelta(pool: Pool, currentPrice: BigDecimal): BigDecimal {
  // Delta for constant product AMM LP position
  // δ = 0.5 * (1 - √(P/P₀))
  let priceRatio = currentPrice.div(pool.initialPrice)
  let sqrtPriceRatio = priceRatio.sqrt()
  
  return BigDecimal.fromString("0.5").times(
    BigDecimal.fromString("1").minus(sqrtPriceRatio)
  )
}

export function calculateImpermanentLoss(currentPrice: BigDecimal, initialPrice: BigDecimal): BigDecimal {
  // IL = 2√(P/P₀) / (1 + P/P₀) - 1
  let priceRatio = currentPrice.div(initialPrice)
  let numerator = BigDecimal.fromString("2").times(priceRatio.sqrt())
  let denominator = BigDecimal.fromString("1").plus(priceRatio)
  
  return numerator.div(denominator).minus(BigDecimal.fromString("1"))
}

export function calculateHedgeAmount(delta: BigDecimal, hedgeRatio: BigDecimal): BigDecimal {
  return delta.times(hedgeRatio).neg()  // Opposite sign to neutralize
}
```

### 5.2 Real-time Metrics Updates

```typescript
// src/mappings/metrics.ts
export function updateDeltaNeutralMetrics(pool: Pool, block: ethereum.Block): void {
  let metricsId = pool.id + "-" + block.timestamp.toString()
  let metrics = new DeltaNeutralMetrics(metricsId)
  
  metrics.pool = pool.id
  
  // Calculate current delta exposure
  let currentPrice = getCurrentPrice(pool)
  metrics.lpDelta = calculateLPDelta(pool, currentPrice)
  metrics.hedgeRatio = getOptimalHedgeRatio(pool)
  metrics.impermanentLoss = calculateImpermanentLoss(currentPrice, pool.initialPrice)
  
  // Calculate strategy metrics
  metrics.totalValueLocked = pool.liquidityUSD
  metrics.borrowingCost = getWeightedBorrowingCost(pool)
  metrics.tradingFeeAPY = calculateTradingFeeAPY(pool)
  
  metrics.timestamp = block.timestamp
  metrics.blockNumber = block.number
  
  metrics.save()
}
```

## Phase 6: Testing & Deployment (1 hour)

### 6.1 Local Testing
```bash
# Build subgraph
graph build

# Deploy to local Graph Node
graph create --node http://localhost:8020 eulerswap-mainnet
graph deploy --node http://localhost:8020 --ipfs http://localhost:5001 eulerswap-mainnet
```

### 6.2 Query Testing
```graphql
# Test query for delta-neutral strategies
{
  pools(first: 10, orderBy: totalVolumeUSD, orderDirection: desc) {
    id
    asset0 { symbol }
    asset1 { symbol }
    liquidityUSD
    totalVolumeUSD
    reserve0
    reserve1
  }
  
  deltaNeutralMetrics(
    first: 100
    orderBy: timestamp
    orderDirection: desc
    where: { pool: "0x..." }
  ) {
    lpDelta
    hedgeRatio
    impermanentLoss
    borrowingCost
    tradingFeeAPY
    timestamp
  }
}
```

### 6.3 Production Deployment
```bash
# Deploy to The Graph's hosted service
graph auth --product hosted-service <ACCESS_TOKEN>
graph deploy --product hosted-service <USERNAME>/eulerswap-mainnet
```

## Phase 7: Integration with Parent Project (30 minutes)

### 7.1 Update Data Client
Modify `../euler-delta-neutral/src/data/subgraph_client.py`:

```python
class EulerSubgraphClient:
    def __init__(self):
        self.eulerswap_endpoint = "https://api.thegraph.com/subgraphs/name/<USERNAME>/eulerswap-mainnet"
    
    async def fetch_eulerswap_data(self, start_date, end_date):
        """Fetch real EulerSwap data instead of synthetic generation."""
        query = """
        query GetEulerSwapData($startTime: Int!, $endTime: Int!) {
          swaps(
            where: { timestamp_gte: $startTime, timestamp_lte: $endTime }
            orderBy: timestamp
            orderDirection: asc
            first: 1000
          ) {
            id
            pool { id asset0 { symbol } asset1 { symbol } }
            amount0In
            amount1In
            amount0Out
            amount1Out
            amountUSD
            deltaExposure
            hedgeRequired
            timestamp
          }
          
          deltaNeutralMetrics(
            where: { timestamp_gte: $startTime, timestamp_lte: $endTime }
            orderBy: timestamp
            orderDirection: asc
          ) {
            lpDelta
            hedgeRatio
            impermanentLoss
            borrowingCost
            tradingFeeAPY
            timestamp
          }
        }
        """
        
        # Execute query and return processed data
        return await self._execute_query(query, variables)
```

### 7.2 Update Strategy Integration
Enhance `../euler-delta-neutral/src/strategy/delta_neutral.py` to use real liquidity constraints and borrowing costs from subgraph data.

## Success Criteria

### Technical Requirements
- [x] Subgraph indexes all EulerSwap pools since June 2024
- [x] Real-time swap event processing with delta calculations
- [x] Vault state tracking with borrowing costs and utilization
- [x] GraphQL API providing all data needed for delta-neutral strategies
- [x] Integration with existing backtesting framework

### Data Quality
- Historical data from June 2024 EulerSwap deployment
- Real-time updates with <30 second latency
- Accurate delta and impermanent loss calculations
- Complete vault integration data (borrowing costs, utilization)

### Performance
- Handle high-frequency swap events efficiently
- Support complex GraphQL queries for strategy analysis
- Derived field calculations optimized for real-time updates

## Available Resources

- **Contract Interfaces**: Complete Solidity interfaces in `contracts/interfaces/`
- **Deployment Addresses**: All network deployments in `deployment-addresses.json`
- **Research Context**: Comprehensive background in `research-context.md`
- **Parent Project**: Reference implementation in `../euler-delta-neutral/`

## Timeline

**Total Development Time**: ~6 hours
- Phase 1 (Setup): 30 minutes
- Phase 2 (Contracts): 1 hour  
- Phase 3 (Schema): 45 minutes
- Phase 4 (Handlers): 2 hours
- Phase 5 (Delta Calculations): 1 hour
- Phase 6 (Testing): 1 hour
- Phase 7 (Integration): 30 minutes

## Expected Outcome

A production-ready subgraph providing comprehensive EulerSwap data that enables:

1. **Real Historical Analysis**: Replace synthetic data with actual market conditions
2. **Live Strategy Execution**: Real-time delta tracking and rebalancing signals  
3. **Educational Value**: Demonstrate delta-neutral concepts with real protocol mechanics
4. **Research Platform**: Foundation for further DeFi strategy development

This subgraph will significantly enhance the educational and practical value of the delta-neutral LP backtesting project by providing access to real EulerSwap market data and protocol mechanics.