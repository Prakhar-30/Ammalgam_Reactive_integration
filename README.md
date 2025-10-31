# Ammalgam Liquidation Protection System

## Overview

A cross-chain liquidation protection system for Ammalgam Protocol using Reactive Smart Contracts. The system monitors user positions on Sepolia and provides automated protection against liquidation through event-driven callbacks between Sepolia (Callback Contract) and Lasna Testnet (Reactive Contract).

---

## System Architecture

### Components

1. **Callback Contract** (Sepolia - Same chain as Ammalgam)
   - Reads Ammalgam state using external functions
   - Calculates health metrics (health factor, LTV)
   - Stores user positions and monitoring settings
   - Emits events with position data
   - Executes protection actions

2. **Reactive Smart Contract** (Lasna Testnet)
   - **Stateless** - no storage, pure automation
   - Listens to Callback Contract events
   - Subscribes to Ammalgam `Liquidate` event (Topic0)
   - Manages cron scheduling per user
   - Sends callbacks to trigger protection

3. **Ammalgam Protocol** (Sepolia - Existing)
   - No modifications required
   - Provides data through external functions
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

Automatic Subscriptions:
✓ Subscribe to ALL events from Callback Contract
✓ Subscribe to Ammalgam's Liquidate event (Topic0)
```

---

### Phase 2: User Subscribes to Protection

**User calls `subscribeProtection(cronInterval)` on Callback Contract**

**Parameters:**
- `cronInterval`: Time between checks (12 seconds to 28 hours)

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

3. **Calculate current tick from reserves:**
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
       lastHealthFactor: healthFactor,
       thresholdHealthFactor: 1.2e18, // Default threshold (1.2x)
       lastCheckTimestamp: block.timestamp
   });
   ```

8. **Emit PositionSubscribed event:**
   ```solidity
   event PositionSubscribed(
       address indexed user,
       uint256 cronInterval,           // User's chosen interval
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
   - Decodes: user, cronInterval, healthFactor, thresholdHealthFactor
   - Subscribes user to cron schedule: cronSchedule[cronInterval].push(user)
   - Checks: if (healthFactor < thresholdHealthFactor)
   ```

10. **Immediate action if needed:**
    ```
    If healthFactor < threshold:
        → Send callback: executeProtection(user)
    Else:
        → User subscribed to cron, wait for next interval
    ```

---

### Phase 3: Continuous Cron Monitoring

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
   - Calculate current tick
   - getTickRange()
   - Build InputParams
   - Calculate health metrics
   ```

4. **Emit PositionChecked event:**
   ```solidity
   event PositionChecked(
       address indexed user,
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
   - Decodes: user, healthFactor, thresholdHealthFactor
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
   ```

   **If healthFactor >= threshold:**
   ```
   - Position is healthy
   - Continue monitoring on next cron interval
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
// User subscribes to liquidation protection
function subscribeProtection(uint256 cronInterval) external;

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

// Event listener for Ammalgam Liquidate event
function onAmmalgamLiquidate(bytes memory eventData) external;
```

---

## Event Specifications

### PositionSubscribed
```solidity
event PositionSubscribed(
    address indexed user,
    uint256 cronInterval,           // 12 sec to 28 hours
    uint256 healthFactor,            // Scaled by 1e18 (e.g., 1.5e18 = 1.5x)
    uint256 currentLTV,              // In basis points (e.g., 6000 = 60%)
    uint256 thresholdHealthFactor,   // Protection threshold (e.g., 1.2e18 = 1.2x)
    uint256 collateralInL,           // Total collateral in L assets
    uint256 debtInL,                 // Total debt in L assets
    uint256 timestamp
);
```

### PositionChecked
```solidity
event PositionChecked(
    address indexed user,
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

### Converting Assets to L (Liquidity Assets)

**X to L conversion:**
```solidity
function convertXToL(
    uint256 amountX,
    uint256 sqrtPriceInQ72,
    uint256 activeLiquidityScalerInQ72
) internal pure returns (uint256 amountL) {
    // amountL = (amountX * Q72 * Q72) / (2 * sqrtPrice * scaler)
    return (amountX * Q72 * Q72) / (2 * sqrtPriceInQ72 * activeLiquidityScalerInQ72);
}
```

**Y to L conversion:**
```solidity
function convertYToL(
    uint256 amountY,
    uint256 sqrtPriceInQ72,
    uint256 activeLiquidityScalerInQ72
) internal pure returns (uint256 amountL) {
    // amountL = (amountY * 2 * sqrtPrice) / scaler
    return (amountY * 2 * sqrtPriceInQ72) / activeLiquidityScalerInQ72;
}
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

3. **`getTickRange()`**
   - Returns: `(int16 minTick, int16 maxTick)`
   - Used for LTV calculations with price bounds

4. **Token `balanceOf(user)`**
   - Called on each of 6 token contracts
   - Returns user's share balance

5. **Token `totalSupply()`**
   - Called on each of 6 token contracts
   - Returns total shares for conversion to assets

### No Modifications to Ammalgam

- ✅ Uses only external/public functions
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
    uint256 lastHealthFactor;
    uint256 thresholdHealthFactor;  // Trigger threshold (e.g., 1.2e18)
    uint256 lastCheckTimestamp;
}

mapping(address => Position) public positions;
```

### Reactive Contract (Lasna) - STATELESS

- ❌ **NO storage variables**
- ❌ **NO mappings**
- ❌ **NO position data**
- ✅ Pure event processing
- ✅ Cron scheduling (transient)
- ✅ Callback triggering

**Benefits:**
- No dual-state complexity
- Single source of truth (Sepolia)
- Gas efficient on Lasna
- Simple architecture

---

## Protection Strategy

### Default Protection: Partial Debt Repayment

**When triggered (healthFactor < threshold):**

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

## Cron Interval Guidelines

```
cronInterval Options (12 seconds to 28 hours):

High Risk (HF 1.0-1.2):     Every 12-60 seconds
Medium Risk (HF 1.2-1.5):   Every 5-15 minutes
Low Risk (HF > 1.5):        Every 1-6 hours
Very Safe (HF > 2.0):       Every 12-28 hours
```

**Trade-offs:**
- ⚡ Shorter intervals = faster protection, higher costs
- 💰 Longer intervals = lower costs, higher risk
- Users choose based on risk tolerance and position size

---


### Price Manipulation Prevention

- ✅ Uses Ammalgam's built-in TWAP (Time-Weighted Average Price)
- ✅ `getTickRange()` includes historical price data
- ✅ Never relies on spot price alone
- ✅ Min/max tick bounds prevent manipulation

### Pre-Authorization Required

```
Users must:
1. Approve tokens to Callback Contract, OR
2. Deposit funds to Callback Contract

Protection fails gracefully if insufficient funds
```

---

---

**Version:** 1.0  
**Last Updated:** October 31, 2025  
**Status:** Proof of Concept Design
