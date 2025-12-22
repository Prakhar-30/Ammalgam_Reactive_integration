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

| Risk Level | Interval | REACT/month | ETH/month | Total (USD estimate) |
|------------|----------|-------------|-----------|---------------------|
| High-Risk  | 12 min   | 12.66      | 8.89      | ~$8,900 @ $1,000/ETH |
| Medium-Risk| 2 hrs    | 1.23       | 0.887     | ~$890 @ $1,000/ETH |
| Moderate-Risk | 28 hrs | 0.09       | 0.063     | ~$63 @ $1,000/ETH |

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

#### Cost Distribution Models

**Model A: Equal Interval Distribution**
```
Assume X users distributed across intervals:
- 30% on 12-minute interval (high-risk)
- 50% on 2-hour interval (medium-risk)
- 20% on 28-hour interval (moderate-risk)

Note: Available intervals are 7s, 1min, 12min, 2hrs, 28hrs
This distribution assumes typical risk profiles across user base.

For X = 100 users:
- 12-min group: 30 users, 120 events/day
  Cost: (120 × 0.0033) + callbacks = 0.396 REACT/day
  Per user: 0.013 REACT/day = 0.40 REACT/month

- 2-hour group: 50 users, 12 events/day
  Cost: (12 × 0.0033) + callbacks = 0.040 REACT/day
  Per user: 0.0008 REACT/day = 0.02 REACT/month

- 28-hour group: 20 users, 0.857 events/day
  Cost: (0.857 × 0.0033) + callbacks = 0.003 REACT/day
  Per user: 0.00015 REACT/day = 0.005 REACT/month
```

**Model B: Batched Processing Efficiency**
```
If Reactive Contract processes users in batches:
- 10 users per cron callback
- Single event loop processes multiple checkPosition() calls

Efficiency gain: ~40% reduction in per-user costs
- High-risk (12m): 0.40 × 0.6 = 0.24 REACT/month
- Medium-risk (2h): 0.02 × 0.6 = 0.012 REACT/month
- Moderate-risk (28h): 0.005 × 0.6 = 0.003 REACT/month
```

### Callback Contract Operations (ETH)

#### Shared Contract Benefits
```
Gas optimization through:
1. Warm storage slots (frequently accessed)
2. Batched state updates
3. Optimized event emissions
4. Cached Ammalgam data

Estimated savings: 15-25% per operation
```

#### Adjusted Gas Costs
```
subscribeProtection() (warm):
- Original: 145,000 gas
- Optimized: ~120,000 gas
- With surcharge: 141,000 total
- Cost: 0.00282 ETH

checkPosition() (warm):
- Original: 102,000 gas
- Optimized: ~85,000 gas
- With surcharge: 106,000 total
- Cost: 0.00212 ETH

executeProtection() (warm):
- Original: 212,000 gas
- Optimized: ~180,000 gas
- With surcharge: 201,000 total
- Cost: 0.00402 ETH
```

#### Per-User Monthly Costs (Platform Model)

**High-Risk User (12-min interval):**
```
REACT: 0.24 REACT/month
ETH: (3,600 × 0.00212) + (6 × 0.00402)
   = 7.63 + 0.02 = 7.65 ETH/month
```

**Medium-Risk User (2-hour interval):**
```
REACT: 0.012 REACT/month
ETH: (360 × 0.00212) + (0.12 × 0.00402)
   = 0.763 + 0.0005 = 0.764 ETH/month
```

**Moderate-Risk User (28-hour interval):**
```
REACT: 0.003 REACT/month
ETH: (25.71 × 0.00212) + (0.003 × 0.00402)
   = 0.055 + 0.00001 = 0.055 ETH/month
```

### Platform-Wide Costs for X Users

**Total Monthly Platform Costs:**

| Users (X) | REACT/month | ETH/month | Total Cost Estimate |
|-----------|-------------|-----------|-------------------|
| 100       | ~4         | ~260      | ~$260K @ $1,000/ETH |
| 500       | ~20        | ~1,300    | ~$1.3M @ $1,000/ETH |
| 1,000     | ~40        | ~2,600    | ~$2.6M @ $1,000/ETH |
| 5,000     | ~200       | ~13,000   | ~$13M @ $1,000/ETH |

**Per-User Platform Cost:**

| Risk Level | Interval | REACT/month | ETH/month | Total (USD) |
|------------|----------|-------------|-----------|------------|
| High-Risk  | 12 min   | 0.24       | 7.65      | ~$7,650    |
| Medium-Risk| 2 hrs    | 0.012      | 0.764     | ~$764      |
| Moderate-Risk | 28 hrs | 0.003      | 0.055     | ~$55       |

---

## Cost Comparison: Individual vs Platform

### Savings with Platform Deployment

| Risk Level | Interval | Individual (ETH/mo) | Platform (ETH/mo) | Savings |
|------------|----------|-------------------|------------------|---------|
| High-Risk  | 12 min   | 8.89             | 7.65             | 13.9%   |
| Medium-Risk| 2 hrs    | 0.887            | 0.764            | 13.9%   |
| Moderate-Risk | 28 hrs | 0.063            | 0.055            | 12.7%   |

**Platform Benefits:**
- 13-14% cost reduction per user
- Shared infrastructure overhead
- Optimized gas usage (warm storage)
- Economies of scale for large user bases

**Individual Benefits:**
- Complete control over monitoring intervals
- Customizable protection thresholds
- No platform dependency
- Privacy preservation

---

## Cost Optimization Strategies

### 1. Dynamic Interval Adjustment
```
Adjust monitoring frequency based on health:
- HF < 1.5: Use 12-minute interval (high-risk)
- HF 1.5-2.0: Use 2-hour interval (medium-risk)
- HF > 2.0: Use 28-hour interval (moderate-risk)

Note: 7-second and 1-minute intervals are also available for
extreme situations requiring more frequent monitoring.

Potential savings: Up to 99% when health improves
Example: User starts at 12m, moves to 28h as health improves
  Savings: 8.89 - 0.063 = 8.83 ETH/month (99.3% reduction)
```

### 2. Threshold-Based Monitoring
```
Only check when:
- Ammalgam price changes > 5%
- User position modified
- Market volatility spikes

Reduces unnecessary checks: 30-50% cost reduction
```

### 3. Batched Operations
```
Platform model can batch:
- Multiple user checks in single callback
- Aggregated health calculations
- Shared Ammalgam state reads

Gas savings: 20-35% per operation
```

### 4. Selective Protection
```
Users choose protection level:
- High monitoring: 12-minute interval
- Balanced: 2-hour interval  
- Conservative: 28-hour interval
- Emergency only: Liquidation event monitoring

Note: 7-second and 1-minute intervals available for extreme cases.

Cost range: 0.063 to 8.89 ETH/month (individual)
           0.055 to 7.65 ETH/month (platform)
```

---

## Operational Cost Summary

### Per-User Costs (Monthly)

**Individual Deployment:**
- High-risk (12m): 12.66 REACT + 8.89 ETH (~$8.9K)
- Medium-risk (2h): 1.23 REACT + 0.887 ETH (~$890)
- Moderate-risk (28h): 0.09 REACT + 0.063 ETH (~$63)

**Platform Deployment:**
- High-risk (12m): 0.24 REACT + 7.65 ETH (~$7.7K)
- Medium-risk (2h): 0.012 REACT + 0.764 ETH (~$760)
- Moderate-risk (28h): 0.003 REACT + 0.055 ETH (~$55)

**Note:** Additional intervals available: 7 seconds and 1 minute for extreme situations.

### Key Takeaways

1. **Callback Contract (ETH) dominates costs** (>99% of total)
2. **Reactive Contract (REACT)** is relatively inexpensive
3. **Platform deployment saves 13-14%** per user
4. **Monitoring frequency is the primary cost driver**
5. **12-minute interval is practical** for high-risk positions (~$8.9K/month)
6. **2-hour and 28-hour intervals are affordable** for most users (<$900/month)
7. **Dynamic interval adjustment can save 99%** of costs

### Recommendations

**For Individual Users:**
- **High-risk positions (HF 1.0-1.5)**: Use 12-minute interval
- **Medium-risk positions (HF 1.5-2.0)**: Use 2-hour interval
- **Moderate-risk positions (HF > 2.0)**: Use 28-hour interval
- Consider 7-second or 1-minute only for critical, high-value positions
- Monitor health factor and adjust interval as position improves

**For Platform Deployment:**
- Implement tiered pricing: High (~$7.7K), Medium (~$760), Moderate (~$55)
- Encourage 2-hour and 28-hour intervals for cost efficiency
- Offer dynamic interval adjustment as premium feature
- Provide health factor monitoring dashboards
- Reserve 7s/1m intervals for emergency subscriptions only
- Consider subsidizing 28-hour monitoring for user adoption

---

**Note:** All estimates assume:
- Sepolia base gas: 20 gwei
- ETH price: $1,000 (for reference only)
- REACT token pricing: Variable market rate
- No network congestion or gas spikes
- Average callback execution rates

Actual costs will vary based on real-time network conditions, user behavior, and market volatility.
