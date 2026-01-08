# Ammalgam Liquidation Protection System

## Overview

A cross-chain liquidation protection system for Ammalgam Protocol using Reactive Smart Contracts. The system monitors user positions on Sepolia and provides automated protection against liquidation through **dual monitoring**: event-driven price monitoring and time-based cron scheduling between Sepolia (Callback Contract) and Lasna Testnet (Reactive Contract).

**Key Enhancement:** The system now monitors Ammalgam Swap events in real-time to detect price movements and trigger protection when price volatility thresholds are exceeded, in addition to periodic time-based health checks.

---

## System Architecture

### Components

1. **Callback Contract** (Same chain as Ammalgam)
   - Reads Ammalgam state using external functions
   - Calculates health metrics (health factor, LTV)
   - Stores user positions and monitoring settings
   - Emits events with position data and price updates
   - Executes protection actions

2. **Reactive Smart Contract** 
   - Listens to Callback Contract events
   - **NEW:** Subscribes to Ammalgam `Swap` event for real-time price monitoring
   - Subscribes to Ammalgam `Liquidate` event (Topic0)
   - Manages cron scheduling per user
   - Tracks price movements and volatility
   - Sends callbacks to trigger protection

3. **Ammalgam Protocol** 
   - No modifications required
   - Provides data through external functions
   - Emits `Swap` event on every swap transaction
   - Emits `Liquidate` event for liquidations

---

## System Flow

### Phase 1: Contract Deployment

**Step 1: Deploy Callback Contract**
```
User → Deploy Callback Contract (Sepolia)

Constructor Parameters:
- ammalgamPairAddress: Address of Ammalgam pair contract
- tokenAddresses[6]: Addresses of 6 Ammalgam tokens
  [depositL, depositX, depositY, borrowL, borrowX, borrowY]
```

**Step 2: Deploy Reactive Contract**
```
User → Deploy Reactive Contract (Lasna Testnet)

Constructor Parameters:
- callbackContractAddress: Address of Callback Contract (Sepolia)
- ammalgamPairAddress: Address of Ammalgam Pair (Sepolia)

Automatic Subscriptions:
✓ Subscribe to ALL required events from Callback Contract
✓ Subscribe to Ammalgam's Swap event
✓ Subscribe to Ammalgam's Liquidate event
```

---

### Phase 2: User Subscribes to Protection

**User calls `subscribeProtection(cronInterval, priceMovementThreshold)` on Callback Contract**

**Parameters:**
- `cronInterval`: Time between checks (12 minutes,2hrs or 28 hours)
- `priceMovementThreshold`: Price movement % to trigger check (e.g., 200 = 2%)

**Process:**

1. **Callback Contract reads Ammalgam state:**
   ```solidity
   // Get reserves
   (reserveX, reserveY, timestamp) = ammalgamPair.getReserves();
   
   // Get total assets across all tokens
   uint128[6] totalAssets = ammalgamPair.totalAssets();
   // Returns: [depositL, depositX, depositY, borrowL, borrowX, borrowY]
   ```

2. **Get user's position:**
   ```solidity
   // For each of 6 tokens, get user's shares
   for (i = 0; i < 6; i++) {
       userShares[i] = tokenContracts[i].balanceOf(user);
       totalShares[i] = tokenContracts[i].totalSupply();
       
       // Convert shares to assets
       userAssets[i] = (userShares[i] * totalAssets[i]) / totalShares[i];
   }
   ```

3. **Calculate current tick and price from reserves:**
   ```solidity
   uint256 priceInQ128 = (reserveX * Q128) / reserveY;
   int16 currentTick = TickMath.getTickAtPrice(priceInQ128);
   ```

4. **Get tick range for LTV calculations:**
   ```solidity
   (int16 minTick, int16 maxTick) = ammalgamPair.getTickRange();
   ```

5. **Build InputParams struct:**
   ```solidity
   InputParams memory params = InputParams({
       userAssets: userAssets,  // [6] array
       sqrtPriceMinInQ72: TickMath.getSqrtPriceAtTick(minTick),
       sqrtPriceMaxInQ72: TickMath.getSqrtPriceAtTick(maxTick),
       activeLiquidityScalerInQ72: sqrt(reserveX * reserveY) * Q72 / activeLiquidityAssets,
       activeLiquidityAssets: totalAssets[DEPOSIT_L] - totalAssets[BORROW_L],
       reservesXAssets: reserveX,
       reservesYAssets: reserveY
   });
   ```

6. **Calculate health metrics:**
   ```solidity
   // Convert X and Y assets to L (liquidity) assets
   netDepositedXinL = convertXToL(userAssets[DEPOSIT_X], params.sqrtPriceMaxInQ72, ...);
   netDepositedYinL = convertYToL(userAssets[DEPOSIT_Y], params.sqrtPriceMinInQ72, ...);
   netBorrowedXinL = convertXToL(userAssets[BORROW_X], params.sqrtPriceMinInQ72, ...);
   netBorrowedYinL = convertYToL(userAssets[BORROW_Y], params.sqrtPriceMaxInQ72, ...);
   
   // Calculate total collateral and debt in L
   collateralInL = (netDepositedXinL + netDepositedYinL) / 2;
   debtInL = (netBorrowedXinL + netBorrowedYinL) / 2;
   
   // Health Factor = (collateral * LTVMAX) / debt
   // LTVMAX = 9000 (90% in basis points)
   healthFactor = (collateralInL * 9000) / debtInL;
   
   // LTV = debt / collateral
   currentLTV = (debtInL * 10000) / collateralInL; // in bips
   ```

7. **Store position in Callback Contract:**
   ```solidity
   positions[user] = Position({
       isActive: true,
       cronInterval: cronInterval,
       priceMovementThreshold: priceMovementThreshold,  // NEW
       lastHealthFactor: healthFactor,
       lastPrice: priceInQ128,  // NEW - baseline price
       thresholdHealthFactor: 1.2e18, // Default threshold (1.2x)
       lastCheckTimestamp: block.timestamp
   });
   ```

8. **Emit PositionSubscribed event:**
   ```solidity
   event PositionSubscribed(
       address indexed user,
       uint256 cronInterval,           // User's chosen interval
       uint256 priceMovementThreshold,  // NEW - Price % threshold (200 = 2%)
       uint256 currentPrice,            // NEW - Current price in Q128
       uint256 healthFactor,            // Current health factor (e.g., 1.5e18 = 1.5)
       uint256 currentLTV,              // Current LTV in bips (e.g., 6000 = 60%)
       uint256 thresholdHealthFactor,   // Threshold to trigger protection (e.g., 1.2e18)
       uint256 collateralInL,           // Total collateral in L assets
       uint256 debtInL,                 // Total debt in L assets
       uint256 timestamp
   );
   ```

9. **Reactive Contract receives event:**
   ```
   - Decodes: user, cronInterval, priceMovementThreshold, currentPrice, healthFactor, thresholdHealthFactor
   - Subscribes user to cron schedule: cronSchedule[cronInterval].push(user)
   - Stores baseline price: userBaselinePrice[user] = currentPrice  // NEW
   - Checks: if (healthFactor < thresholdHealthFactor)
   ```

10. **Immediate action if needed:**
    ```
    If healthFactor < threshold:
        → Send callback: executeProtection(user)
    Else:
        → User subscribed to dual monitoring:
           1. Cron-based: wait for next interval
           2. Price-based: monitor Swap events
    ```

---

### Phase 3A: Real-Time Price Monitoring (NEW)

**Reactive Contract monitors Ammalgam Swap events continuously**

**Swap Event Structure:**
```solidity
event Swap(
    address indexed sender,      // Address initiating the swap
    uint256 amountXIn,          // Amount of token X provided
    uint256 amountYIn,          // Amount of token Y provided
    uint256 amountXOut,         // Amount of token X received
    uint256 amountYOut,         // Amount of token Y received
    address indexed to          // Recipient address
);
```

**Flow:**

1. **Ammalgam emits Swap event:**
   ```
   Any swap on specific Ammalgam Pair→ Swap event emitted
   ```

2. **Reactive Contract detects Swap event:**
   ```
   Reactive Contract receives Swap event
   → Event is triggered on EVERY swap transaction
   ```

3. **Reactive Contract sends callback for price update:**
   ```
   Reactive Contract → Callback Contract: updatePrice()
   
   Purpose: Read current pool state
   ```

4. **Callback Contract reads current pool state:**
   ```solidity
   // Get current reserves to calculate price
   (reserveX, reserveY, timestamp) = ammalgamPair.getReserves();
   ```

5. **Callback Contract emits PriceUpdated event:**
   ```solidity
   event PriceUpdated(
       uint256 reserveX,           // Current reserve X
       uint256 reserveY,           // Current reserve Y
       uint256 timestamp
   );
   ```

6. **Reactive Contract receives PriceUpdated event:**
   ```
   - Decodes: currentPrice, reserveX, reserveY
   - Calculate current price
       - currentPriceInQ128 = (reserveX * Q128) / reserveY;
   - For each monitored user:
       - Compare to baseline: userBaselinePrice[user]
       - Calculate price movement %
   ```

7. **Reactive Contract checks price movement threshold:**
   ```javascript
   For each user with active protection:
   
   priceMovement = abs(currentPrice - baselinePrice) * 10000 / baselinePrice;
   
   if (priceMovement >= user.priceMovementThreshold) {
       → Price threshold exceeded!
       → Send callback: checkPosition(user)
   }
   ```

8. **Position check triggered by price movement:**
   ```
   If price movement threshold exceeded:
       Reactive Contract → Callback Contract: checkPosition(user)
       → Triggers protection
       → Updates baseline price after check
   ```

### Phase 3B: Continuous Cron Monitoring (Time-Based)

**Every cron interval (user-specific), Reactive Contract triggers:**

1. **Get users for this interval:**
   ```
   Reactive Contract: users = cronSchedule[cronInterval]
   ```

2. **For each user, send callback:**
   ```
   Reactive Contract → Callback Contract: checkPosition(user)
   ```

3. **Callback Contract reads Ammalgam state:**
   ```solidity
   // Same process as subscription:
   - getReserves()
   - totalAssets()
   - balanceOf(user) for each token
   - Convert shares to assets
   - Calculate current tick and price
   - getTickRange()
   - Build InputParams
   - Calculate health metrics
   ```

4. **Emit PositionChecked event:**
   ```solidity
   event PositionChecked(
       address indexed user,
       uint256 currentPrice,            // NEW - Current price
       uint256 healthFactor,
       uint256 currentLTV,
       uint256 thresholdHealthFactor,
       uint256 collateralInL,
       uint256 debtInL,
       uint256 timestamp
   );
   ```

5. **Reactive Contract receives event:**
   ```
   - Decodes: user, currentPrice, healthFactor, thresholdHealthFactor
   - Updates baseline price: userBaselinePrice[user] = currentPrice  // NEW
   - Checks: if (healthFactor < thresholdHealthFactor)
   ```

6. **Protection execution if threshold met:**

   **If healthFactor < threshold:**
   ```
   Reactive Contract → Callback Contract: executeProtection(user)
   ```

   **Callback Contract executes protection:**
   ```solidity
   // Read latest state atomically
   - getReserves(), totalAssets(), balanceOf()
   - Recalculate current health factor
   
   // Calculate repayment needed
   targetHealthFactor = 1.5e18; // Target 1.5x health
   repayAmount = currentDebt - (collateral * LTVMAX / targetHealthFactor);
   
   // Execute repayment (user must have pre-approved tokens)
   ammalgamPair.repay(user, repayAmount, assetType);
   
   // Ammalgam validates solvency and updates saturation
   // validateOnUpdate(user, user, true) is called internally
   ```

   **Emit ProtectionExecuted event:**
   ```solidity
   event ProtectionExecuted(
       address indexed user,
       uint256 repaidAmount,
       uint256 repaidAssetType,      // 3=borrowL, 4=borrowX, 5=borrowY
       uint256 oldHealthFactor,
       uint256 newHealthFactor,
       uint256 gasUsed,
       uint256 timestamp
   );
   ```

   **Reactive Contract receives event:**
   ```
   - Stateless processing: logs success
   - Continues monitoring on next cron interval
   - Continues monitoring Swap events for price changes
   ```

   **If healthFactor >= threshold:**
   ```
   - Position is healthy
   - Continue dual monitoring:
     1. Next cron interval
     2. Swap event monitoring
   ```

---

### Phase 4: Ammalgam Liquidate Event Monitoring

**Reactive Contract is subscribed to Ammalgam's Liquidate event (Topic0)**

**When liquidation occurs:**

1. **Ammalgam emits Liquidate event:**
   ```solidity
   event Liquidate(
       address indexed borrower,
       address indexed to,           // Liquidator
       uint256 depositL,
       uint256 depositX,
       uint256 depositY,
       uint256 repayLX,
       uint256 repayLY,
       uint256 repayX,
       uint256 repayY,
       uint256 liquidationType       // 0=HARD, 1=SOFT, 2=LEVERAGE
   );
   ```

2. **Reactive Contract detects event:**
   ```
   - Decodes: borrower, liquidationType, amounts
   - Determines if borrower is a monitored user
   ```

3. **Send callback to verify position:**
   ```
   Reactive Contract → Callback Contract: checkPosition(borrower)
   ```

4. **Callback Contract checks post-liquidation state:**
   ```
   - Reads updated position from Ammalgam
   - Calculates remaining health factor
   - Emits PositionChecked event
   ```

5. **Reactive Contract evaluates:**
   ```
   If position still at risk:
       → Send callback: executeProtection(borrower)
       → Attempt to save remaining position
   
   If position closed or healthy:
       → Monitoring continues or ends
   ```

---

## Key Functions

### Callback Contract Functions

```solidity
// User subscribes to liquidation protection with dual monitoring
function subscribeProtection(
    uint256 cronInterval,
    uint256 priceMovementThreshold  // NEW - in basis points (200 = 2%)
) external;

// Update current price from pool state (called on Swap events)
function updatePrice() external;  // NEW

// Check position health (called by Reactive Contract)
function checkPosition(address user) external;

// Execute protection (called by Reactive Contract)
function executeProtection(address user) external;

// User unsubscribes from protection
function unsubscribeProtection() external;
```

### Reactive Contract Functions

```solidity
// Cron trigger - processes all users for this interval
function cronCallback(uint256 interval) external;

// Event listener for Callback Contract events
function onCallbackEvent(bytes memory eventData) external;

// NEW: Event listener for Ammalgam Swap events
function onAmmalgamSwap(bytes memory eventData) external;

// Event listener for Ammalgam Liquidate event
function onAmmalgamLiquidate(bytes memory eventData) external;
```

---

## Event Specifications

### PositionSubscribed (Enhanced)
```solidity
event PositionSubscribed(
    address indexed user,
    uint256 cronInterval,           // 12 sec to 28 hours
    uint256 priceMovementThreshold,  // NEW - In basis points (200 = 2%)
    uint256 currentPrice,            // NEW - Baseline price in Q128
    uint256 healthFactor,            // Scaled by 1e18 (e.g., 1.5e18 = 1.5x)
    uint256 currentLTV,              // In basis points (e.g., 6000 = 60%)
    uint256 thresholdHealthFactor,   // Protection threshold (e.g., 1.2e18 = 1.2x)
    uint256 collateralInL,           // Total collateral in L assets
    uint256 debtInL,                 // Total debt in L assets
    uint256 timestamp
);
```

### PriceUpdated (NEW)
```solidity
event PriceUpdated(
    uint256 currentPrice,       // Current price in Q128 format
    uint256 reserveX,           // Current reserve X
    uint256 reserveY,           // Current reserve Y
    uint256 timestamp
);
```

### PositionChecked (Enhanced)
```solidity
event PositionChecked(
    address indexed user,
    uint256 currentPrice,            // NEW - Current price
    uint256 healthFactor,
    uint256 currentLTV,
    uint256 thresholdHealthFactor,
    uint256 collateralInL,
    uint256 debtInL,
    uint256 timestamp
);
```

### ProtectionExecuted
```solidity
event ProtectionExecuted(
    address indexed user,
    uint256 repaidAmount,
    uint256 repaidAssetType,        // 3=borrowL, 4=borrowX, 5=borrowY
    uint256 oldHealthFactor,
    uint256 newHealthFactor,
    uint256 gasUsed,
    uint256 timestamp
);
```

---

## Health Factor & LTV Calculations

### Health Factor Formula
```
healthFactor = (collateralValue * LTVMAX) / debtValue

Where:
- LTVMAX = 9000 (90% in basis points)
- healthFactor > 1.0 means position is safe
- healthFactor < 1.0 means position is liquidatable
- healthFactor = 1.2 is common threshold for protection
```

### LTV (Loan-to-Value) Formula
```
LTV = (debtValue * 10000) / collateralValue

Expressed in basis points:
- LTV = 6000 means 60%
- LTV = 7500 means 75%
- LTV > 9000 (90%) is liquidatable
```

### Price Calculation
```solidity
// Price in Q128 format (fixed-point)
price = (reserveX * Q128) / reserveY

// Price movement calculation
priceMovement = abs(currentPrice - baselinePrice) * 10000 / baselinePrice
// Result in basis points (200 = 2%)
```

---

## Ammalgam Data Access

### External Functions Used

All data is obtained using Ammalgam's existing external functions:

1. **`totalAssets()`**
   - Returns: `uint128[6]` - Total assets for all 6 token types
   - Indices: 0=depositL, 1=depositX, 2=depositY, 3=borrowL, 4=borrowX, 5=borrowY

2. **`getReserves()`**
   - Returns: `(uint112 reserveX, uint112 reserveY, uint32 lastTimestamp)`
   - **NEW:** Used to calculate price after Swap events

3. **`getTickRange()`**
   - Returns: `(int16 minTick, int16 maxTick)`
   - Used for LTV calculations with price bounds

4. **Token `balanceOf(user)`**
   - Called on each of 6 token contracts
   - Returns user's share balance

5. **Token `totalSupply()`**
   - Called on each of 6 token contracts
   - Returns total shares for conversion to assets

### Events Monitored

1. **`Swap` Event (NEW)**
   ```solidity
   event Swap(
       address indexed sender,
       uint256 amountXIn,
       uint256 amountYIn,
       uint256 amountXOut,
       uint256 amountYOut,
       address indexed to
   );
   ```
   - **Purpose:** Detect price movements in real-time
   - **Frequency:** Emitted on every swap transaction
   - **Action:** Trigger price update and threshold check

2. **`Liquidate` Event**
   - **Purpose:** Detect when users get liquidated
   - **Frequency:** Emitted on liquidation events
   - **Action:** Check if monitored user, attempt recovery

### No Modifications to Ammalgam

- ✅ Uses only external/public functions
- ✅ Uses only existing events (Swap, Liquidate)
- ✅ No new functions added (respects code size limits)
- ✅ Replicates internal logic externally
- ✅ Compatible with existing Ammalgam contracts

---

## State Management

### Callback Contract (Sepolia) - STATEFUL

Stores all position data:
```solidity
struct Position {
    bool isActive;
    uint256 cronInterval;           // User's chosen check interval
    uint256 priceMovementThreshold;  // NEW - Price % threshold (in bips)
    uint256 lastPrice;               // NEW - Last known price (Q128)
    uint256 lastHealthFactor;
    uint256 thresholdHealthFactor;  // Trigger threshold (e.g., 1.2e18)
    uint256 lastCheckTimestamp;
}

mapping(address => Position) public positions;
```

**Benefits:**
- No dual-state complexity
- Single source of truth (Sepolia)
- Gas efficient on Lasna
- Simple architecture

---

## Protection Strategy

### Default Protection: Partial Debt Repayment

**When triggered (healthFactor < threshold OR priceMovement > threshold):**

1. **Calculate target repayment:**
   ```solidity
   targetHealthFactor = 1.5e18; // Target 1.5x for safety buffer
   
   // Current: HF = (collateral * 9000) / debt
   // Target:  1.5 = (collateral * 9000) / newDebt
   // Solve:   newDebt = (collateral * 9000) / 1.5
   
   repayAmount = currentDebt - newDebt;
   ```

2. **Determine optimal asset:**
   ```
   - Repay borrowL if available
   - Otherwise repay borrowX or borrowY
   - Choose based on lowest slippage
   ```

3. **Execute repayment:**
   ```
   User must have:
   - Pre-approved tokens to Callback Contract, OR
   - Deposited funds in Callback Contract
   
   Callback Contract calls Ammalgam on behalf of user
   ```

4. **Verification:**
   ```
   - Ammalgam internally calls validateOnUpdate()
   - Saturation is updated automatically
   - New health factor is calculated
   ```

---

## Monitoring Guidelines

### Recommended Configurations

```
Position Risk Level → Configuration

CRITICAL (HF 1.0-1.15):
├─ Cron: Every 12 seconds
├─ Price Threshold: 50 bips (0.5%)
└─ Max Protection: Fastest response, highest cost

HIGH RISK (HF 1.15-1.3):
├─ Cron: Every 1 minute
├─ Price Threshold: 100 bips (1%)
└─ Strong Protection: Fast response, high cost

MEDIUM RISK (HF 1.3-1.5):
├─ Cron: Every 12 minutes
├─ Price Threshold: 200 bips (2%)
└─ Balanced: Good protection, moderate cost

LOW RISK (HF 1.5-2.0):
├─ Cron: Every 2 hours
├─ Price Threshold: 300 bips (3%)
└─ Cost Efficient: Basic protection, low cost

VERY SAFE (HF > 2.0):
├─ Cron: Every 28 hours
├─ Price Threshold: 500 bips (5%)
└─ Minimum Cost: Emergency protection only
```

---

## Security Features

### Price Manipulation Prevention

- ✅ Uses Ammalgam's built-in TWAP (Time-Weighted Average Price)
- ✅ `getTickRange()` includes historical price data
- ✅ Never relies on spot price alone for protection decisions
- ✅ Min/max tick bounds prevent manipulation
- ✅ Dual monitoring prevents gaming single trigger mechanism

### Pre-Authorization Required

```
Users must:
1. Approve tokens to Callback Contract, OR
2. Deposit funds to Callback Contract

Protection fails gracefully if insufficient funds
```

### Swap Event Monitoring Safety

```
✅ Monitors ALL swaps, not just user transactions
✅ Price updates are atomic with reserve reads
✅ Threshold checks prevent false triggers from minor fluctuations
✅ Baseline price updates after successful checks
```

---

## Flow Summary

### Complete Protection Flow

```
User Subscribes
    ↓
[Dual Monitoring Active]
    ├─→ Time-Based (Cron)
    │   └─→ Every X interval → checkPosition()
    │
    └─→ Price-Based (Swap Events)
        └─→ On every Swap → updatePrice()
            └─→ If movement > threshold → checkPosition()
    
checkPosition()
    ↓
Health Factor < Threshold?
    ├─→ YES: executeProtection()
    │   └─→ Repay debt
    │       └─→ Update baseline price
    │
    └─→ NO: Continue monitoring
        └─→ Update baseline price

Liquidate Event Detected
    ↓
checkPosition() → Verify state
    ↓
Attempt recovery if needed
```

---

**Version:** 2.0 (Enhanced with Price Monitoring)  
