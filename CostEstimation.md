# Gas Cost Analysis: Ammalgam Liquidation Protection System

## Overview

This document analyzes operational gas costs for the liquidation protection system across two deployment scenarios:
1. **Individual Deployment**: Each user deploys their own contracts
2. **Platform Deployment**: Ammalgam deploys shared contracts for all users

---

## Gas Cost Formulas

## Reactive Contract (Lasna Testnet)
```
RVM Transaction Fee = BaseFee × GasUsed

Where:
- BaseFee: Base fee per gas unit (block n+1)
- GasUsed: Actual gas consumed
- Max gas limit: 900,000 units per RVM transaction
- Unit: REACT tokens
```

### Callback Contract (Sepolia)
```
Callback Cost = Pbase × C × (Gcallback + K)

Where:
- Pbase: Base gas price (tx.gasprice or block.basefee)
- C: Pricing coefficient for Sepolia
- Gcallback: Gas consumed during callback execution
- K: Fixed gas surcharge for Sepolia
- Unit: ETH
```

---

## Scenario 1: Individual Deployment (Per User)

### Reactive Contract Operations (REACT tokens)

#### 1. Event Listening (Passive Monitoring)
```
Operation: Listen to cron events without callbacks
Cost: ~0.0033 REACT per cron interval

Available Cron Intervals and Costs:
- Every 7 seconds: 12,343 events/day → ~40.732 REACT/day
- Every 1 minute: 1,440 events/day → ~4.752 REACT/day
- Every 12 minutes: 120 events/day → ~0.396 REACT/day
- Every 2 hours: 12 events/day → ~0.040 REACT/day
- Every 28 hours: 0.857 events/day → ~0.003 REACT/day
```

#### 2. Event Listening + Callback Emission
```
Operation: Listen to events and emit callbacks
Cost: 0.0033 REACT (base) + Variable (callback size)

Callback Size Impact:
- Small callback (<100 bytes): +0.0010 REACT
- Medium callback (100-500 bytes): +0.0025 REACT
- Large callback (>500 bytes): +0.0050 REACT

Total per event with callback:
- Small: ~0.0043 REACT
- Medium: ~0.0058 REACT
- Large: ~0.0083 REACT
```

#### 3. Position Monitoring Scenarios

**Note on Cron Intervals:**
```
Available Cron Intervals: 7 seconds, 1 minute, 12 minutes, 2 hours, 28 hours

The following scenarios use assumed interval selections based on risk levels.
In practice, users can choose any available interval regardless of their position's
health factor. These are recommended mappings for typical use cases.
```

**High-Risk Position (HF 1.0-1.5)**
- Interval: Every 12 minutes (120 checks/day)
- Callback rate: 5% (frequent threshold approaches)
- Cost calculation:
  ```
  Daily = (120 × 0.0033) + (6 × 0.0043)
        = 0.396 + 0.026
        = 0.422 REACT/day
        = 12.66 REACT/month
  ```

**Medium-Risk Position (HF 1.5-2.0)**
- Interval: Every 2 hours (12 checks/day)
- Callback rate: 1% (occasional threshold approaches)
- Cost calculation:
  ```
  Daily = (12 × 0.0033) + (0.12 × 0.0043)
        = 0.040 + 0.001
        = 0.041 REACT/day
        = 1.23 REACT/month
  ```

**Moderate-Risk Position (HF > 2.0)**
- Interval: Every 28 hours (0.857 checks/day)
- Callback rate: 0.1% (very rare callbacks)
- Cost calculation:
  ```
  Daily = (0.857 × 0.0033) + (0.0009 × 0.0043)
        = 0.003 + 0.000
        = 0.003 REACT/day
        = 0.09 REACT/month
  ```

## Callback Contract Operations (ETH)

#### Gas Consumption Estimates

**subscribeProtection()**
```
Operations:
- 1× getReserves() → ~5,000 gas
- 1× totalAssets() → ~8,000 gas
- 6× balanceOf() calls → ~36,000 gas (6,000 each)
- 6× totalSupply() calls → ~18,000 gas (3,000 each)
- 1× getTickRange() → ~5,000 gas
- Share to asset conversion (6 tokens) → ~10,000 gas
- Health calculations → ~15,000 gas
- Storage writes → ~45,000 gas
- Event emission → ~3,000 gas

Total: ~145,000 gas
```

**checkPosition()**
```
Operations:
- 1× getReserves() → ~5,000 gas
- 1× totalAssets() → ~8,000 gas
- 6× balanceOf() calls → ~36,000 gas
- 6× totalSupply() calls → ~18,000 gas
- 1× getTickRange() → ~5,000 gas
- Calculations → ~25,000 gas
- Storage read (warm) → ~2,100 gas
- Event emission → ~3,000 gas

Total: ~102,000 gas
```

**executeProtection()**
```
Operations:
- All checkPosition() operations → ~102,000 gas
- Calculate repayment amount → ~5,000 gas
- External call to Ammalgam repay() → ~80,000 gas
- Storage update → ~20,000 gas
- Event emission → ~5,000 gas

Total: ~212,000 gas
```

#### Cost Calculations (ETH)

**Assuming:**
- Base gas price (Pbase): 20 gwei
- Pricing coefficient (C): 1.0 (Sepolia)
- Fixed surcharge (K): 21,000 gas

**subscribeProtection()**
```
Cost = 20 gwei × 1.0 × (145,000 + 21,000)
     = 20 gwei × 166,000
     = 3,320,000 gwei
     = 0.00332 ETH per subscription
```

**checkPosition() (per check)**
```
Cost = 20 gwei × 1.0 × (102,000 + 21,000)
     = 20 gwei × 123,000
     = 2,460,000 gwei
     = 0.00246 ETH per check
```

**executeProtection() (when triggered)**
```
Cost = 20 gwei × 1.0 × (212,000 + 21,000)
     = 20 gwei × 233,000
     = 4,660,000 gwei
     = 0.00466 ETH per execution
```

#### Monthly Cost Examples

**High-Risk Position (12-minute interval):**
- Checks: 120/day × 30 days = 3,600 checks
- Executions: ~6 times (5% callback rate)
- Cost: (3,600 × 0.00246) + (6 × 0.00466)
       = 8.86 + 0.03 = **8.89 ETH/month**

**Medium-Risk Position (2-hour interval):**
- Checks: 12/day × 30 days = 360 checks
- Executions: ~0.12 times (1% callback rate)
- Cost: (360 × 0.00246) + (0.12 × 0.00466)
       = 0.886 + 0.001 = **0.887 ETH/month**

**Moderate-Risk Position (28-hour interval):**
- Checks: 0.857/day × 30 days = 25.71 checks
- Executions: ~0.003 times (0.1% callback rate)
- Cost: (25.71 × 0.00246) + (0.003 × 0.00466)
       = 0.063 + 0.00001 = **0.063 ETH/month**

### Total Individual User Costs

| Risk Level | Interval | REACT/month | ETH/month | 
|------------|----------|-------------|-----------|
| High-Risk  | 12 min   | 12.66      | 8.89       | 
| Medium-Risk| 2 hrs    | 1.23       | 0.887      | 
| Moderate-Risk | 28 hrs | 0.09       | 0.063     | 

---

## Scenario 2: Platform Deployment (Ammalgam Shared Contracts)

### Architecture
- **One Reactive Contract** on Lasna (shared)
- **One Callback Contract** on Sepolia (shared)
- **X users** subscribe to protection

### Reactive Contract Operations (REACT tokens)

#### Per-User Subscription Cost
```
When user subscribes:
- Listen to PositionSubscribed event: 0.0033 REACT
- Subscribe user to cron: negligible (in-memory)
- Check initial health: 0.0043 REACT (if callback needed)

Total per new user: ~0.0033 to 0.0076 REACT
```

#### Shared Cron Monitoring
```
Platform manages cron schedules:
- Users grouped by interval preference
- Single cron event processes multiple users
- Cost amortized across batch

Example: 100 users on 5-minute interval
- Cron triggers: 288 times/day
- Cost per trigger: 0.0033 REACT (base listening)
- Callbacks: Variable based on user health

Per-user cost = Total cost / Number of users
```
---


