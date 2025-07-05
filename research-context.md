# EulerSwap Research Context & Background

## Overview

This document provides comprehensive background research conducted to discover EulerSwap contract addresses and understand the protocol for subgraph development.

## Key Discoveries

### 1. EulerSwap Protocol Summary

**EulerSwap** is an automated market maker (AMM) that integrates with Euler credit vaults to provide deeper liquidity for swaps. Key features:

- **Just-in-Time Liquidity**: Borrows required output tokens using input tokens as collateral
- **40x Liquidity Depth**: Enables up to 40x the liquidity depth of traditional AMMs
- **Shared Liquidity**: Single cross-collateralized credit vault supports multiple asset pairs
- **Flexible AMM Curve**: Optimizes swap pricing with customizable mechanics
- **Uniswap v4 Integration**: Compatible with Uniswap v4 hooks where deployed

### 2. Architecture Integration

EulerSwap integrates deeply with the Euler protocol ecosystem:

- **Euler Vault Kit (EVK)**: Built on ERC-4626 vault standard
- **Ethereum Vault Connector (EVC)**: Foundational layer for lending markets
- **Credit Vaults**: Provide the underlying liquidity for just-in-time borrowing
- **Lending Protocol**: Native support for LP positions as collateral

### 3. Live Deployment History

**June 2024**: EulerSwap deployed to Ethereum mainnet with $500,000 in real liquidity as part of a live Capture-the-Flag (CTF) security competition on Cantina.xyz. This was not a simulation but live production testing.

**Key Details**:
- USDC/USDT swap contracts on Ethereum mainnet
- $500,000 in real, onchain liquidity
- Six completed audits prior to deployment
- Transparent, high-stakes testing of live contracts

### 4. Network Deployments

EulerSwap is deployed across multiple networks with two variants:

#### With Uniswap v4 Hook Support
- **Ethereum Mainnet** (Chain ID: 1) - Primary deployment
- **Unichain** (Chain ID: 130)
- **Avalanche C-Chain** (Chain ID: 43114)
- **BNB Smart Chain** (Chain ID: 56)
- **Base** (Chain ID: 8453)

#### Without Uniswap v4 Hook (Original "OG" Version)
- **Sonic** (Chain ID: 146)
- **Swellchain** (Chain ID: 1923)
- **Berachain** (Chain ID: 80094)
- **BOB** (Chain ID: 60808)

### 5. Contract Architecture

Each deployment consists of three main contracts:

1. **Factory Contract**: Deploys and manages pool instances
   - `deployPool()`: Creates new pools with deterministic addresses
   - `uninstallPool()`: Removes pools from tracking
   - Events: `PoolDeployed`, `PoolUninstalled`

2. **Implementation Contract**: Core pool logic
   - `swap()`: Main swap function (Uniswap V2 compatible ABI)
   - `computeQuote()`: Price quotation for exact input/output
   - `getLimits()`: Current liquidity constraints
   - `getReserves()`: Pool state and lock status

3. **Periphery Contract**: Helper functions
   - `swapExactIn()` / `swapExactOut()`: User-friendly swap functions
   - `quoteExactInput()` / `quoteExactOutput()`: Quotation helpers
   - Deadline and slippage protection

### 6. Key Events for Subgraph Tracking

Based on interface analysis, important events to track:

- **Factory Events**:
  - `PoolDeployed(address indexed pool, ...)`
  - `PoolUninstalled(address indexed pool)`

- **Pool Events** (inferred from functions):
  - Swap events with amounts and participants
  - Reserve updates and state changes
  - Liquidity limit changes

- **Vault Integration Events**:
  - Borrow/repay events from underlying Euler vaults
  - Collateral state changes
  - Utilization rate updates

### 7. Delta-Neutral Strategy Context

This research was conducted to support a delta-neutral LP backtesting project with these requirements:

- **Real Historical Data**: Replace synthetic data with actual EulerSwap transactions
- **Vault Integration**: Track borrowing costs, utilization rates, and capacity
- **Delta Tracking**: Real-time position delta calculation and rebalancing
- **Strategy Optimization**: Historical analysis since June 2024 deployment
- **Educational Purpose**: Demonstrate delta-neutral concepts with real market data

### 8. Technical Specifications Needed

For subgraph development:

#### Data Requirements
- Pool deployments and configurations
- All swap transactions with amounts and fees
- Vault state changes (liquidity, utilization, rates)
- Real-time delta exposure calculations
- Historical price movements and volume

#### Performance Considerations
- Index from June 2024 deployment block
- Handle high-frequency swap events efficiently
- Calculate derived fields (delta, IL, APY) in real-time
- Support multiple asset pairs and pool configurations

#### Integration Points
- Connect to existing Python backtesting framework
- Replace synthetic data pipeline in `src/data/subgraph_client.py`
- Enhance strategy with real liquidity constraints
- Support historical analysis and strategy optimization

### 9. Repository Structure

**Main Repository**: `euler-xyz/euler-swap`
- Comprehensive test suite in Solidity
- Deployment scripts for all networks
- Complete interface definitions
- Foundry-based development environment

**Periphery Repository**: `euler-xyz/evk-periphery`
- Additional swap handlers and integrations
- Router factory and swap verification contracts
- Extended functionality for EVK ecosystem

### 10. Security & Validation

EulerSwap underwent extensive security review:
- Five leading audit firms involved
- Independent researchers from early development
- Custom fuzzing campaign since January 2024
- Live mainnet CTF with $500,000 at risk
- Continuous bug bounty program

## Next Steps for Subgraph Development

1. **Contract ABI Extraction**: Use provided interfaces to generate GraphQL schema
2. **Event Definition**: Map contract events to subgraph entities
3. **Schema Design**: Structure for pools, swaps, vaults, and derived metrics
4. **Integration**: Connect to existing delta-neutral backtesting framework
5. **Historical Indexing**: Start from June 2024 deployment block
6. **Real-time Processing**: Handle live swap events and vault state changes

This research provides the complete foundation needed for comprehensive EulerSwap subgraph development and integration with delta-neutral LP strategies.