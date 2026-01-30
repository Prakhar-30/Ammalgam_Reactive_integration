# Ammalgam Liquidation Protection System - Updated POC

## Overview

A cross-chain liquidation protection system for Ammalgam Protocol using Reactive Smart Contracts. The system monitors user positions on the origin chain (e.g., Sepolia) and provides automated protection against liquidation through **dual monitoring**: tick-based price monitoring and time-based cron scheduling between the Origin Chain (Callback Contract) and Reactive Network (Reactive Contract).

**Key Updates in Version 3.0:**
- ✅ **Tick-based price monitoring** instead of basis points
- ✅ **Updated to new `totalAssetsAndShares()` function** (replaces `totalAssets()`)
- ✅ **Simplified liquidation interface** with new event structure
- ✅ **Use of `getTickRange()` with reserves** for tick calculations
- ✅ **Integration with LiquidationUtils** for position analysis
- ✅ **Support for partial liquidations** with tranches
- ✅ **Removed activeLiquidityScalerInQ72** (always 1 now)
- ✅ **Whitelist support** for tracked asset pairs

---

## System Architecture

### Components

1. **Callback Contract** (Same chain as Ammalgam)
   - Reads Ammalgam state using external functions
   - Calculates health metrics (health factor, LTV) using tick-based pricing
   - Stores user positions and monitoring settings
   - Emits events with position data and tick updates
   - Executes protection actions (partial liquidations when needed)

2. **Reactive Smart Contract** (Reactive Network)
   - Listens to Callback Contract events
   - **UPDATED:** Subscribes to Ammalgam `Swap` event for real-time tick monitoring
   - Subscribes to Ammalgam `Liquidate` event (new simplified structure)
   - Manages cron scheduling per user
   - Tracks tick movements and volatility
   - Sends callbacks to trigger protection

3. **Ammalgam Protocol** 
   - No modifications required
   - Provides data through external functions
   - Emits `Swap` event on every swap transaction
   - Emits `Liquidate` event for liquidations (NEW simplified format)

4. **Saturation Contract** (Singleton)
   - Responsible for tick range data for all pairs
   - Provides `getTickRange()` with reserve-based calculations

---

## Key Changes

### 1. Tick-Based Price Monitoring (Instead of Basis Points)

```solidity
tickMovementThreshold: 5 ticks
tickMovement = abs(currentTick - baselineTick);
// Smaller thresholds possible: even 1 tick for critical positions
```

### 2. New `totalAssetsAndShares()` Function

```solidity
ITokenController tokenController = ITokenController(ammalgamPair);
(uint112[6] memory allAssets, uint112[6] memory allShares) = 
    tokenController.totalAssetsAndShares(true); // with interest accrued
```

**Benefits:**
- Single call gets both assets and shares with interest
- More efficient and accurate
- Includes protocol fees for DEPOSIT_L/X/Y

### 3. Simplified Liquidation Event

```solidity
event Liquidate(
    address indexed borrower,
    address indexed to,
    uint256 seizedLAssets,        // Seized from deposit
    uint256 seizedXAssets,
    uint256 seizedYAssets,
    uint256 repayXAssets,          // Requested credit amount
    uint256 repayYAssets,
    uint256 actualRepaidXAssets,   // Actual amount repaid
    uint256 actualRepaidYAssets,
    uint256 liquidationType        // HARD=0, SATURATION=1, LEVERAGE=2
);
```


### 4. Updated `getTickRange()` with Reserves

```solidity
ISaturationAndGeometricTWAPState saturationState = 
    ISaturationAndGeometricTWAPState(SATURATION_CONTRACT_ADDRESS);

(uint112 reserveX, uint112 reserveY,) = ammalgamPair.getReserves();

(int16 minTick, int16 maxTick) = saturationState.getTickRange(
    address(ammalgamPair),
    reserveX,
    reserveY,
    true  // includeLongTermTick
);
```

### 5. Tick Calculation from Reserves

**NEW Helper Function:**
```solidity
int16 currentTick = TickMath.getTickFromReserves(reserveX, reserveY);
```

**Implementation:**
```solidity
// In TickMath library
function getTickFromReserves(
    uint256 reserveXAssets, 
    uint256 reserveYAssets
) internal pure returns (int16) {
    return getTickAtPrice(
        Convert.mulDiv(reserveXAssets, Q128, reserveYAssets, false)
    );
}
```

### 6. Removed `activeLiquidityScalerInQ72`

**OLD InputParams:**
```solidity
InputParams memory params = InputParams({
    userAssets: userAssets,
    sqrtPriceMinInQ72: TickMath.getSqrtPriceAtTick(minTick),
    sqrtPriceMaxInQ72: TickMath.getSqrtPriceAtTick(maxTick),
    activeLiquidityScalerInQ72: sqrt(reserveX * reserveY) * Q72 / activeLiquidity, // REMOVED!
    activeLiquidityAssets: totalAssets[DEPOSIT_L] - totalAssets[BORROW_L],
    reservesXAssets: reserveX,
    reservesYAssets: reserveY
});
```

```solidity
InputParams memory params = InputParams({
    userAssets: userAssets,
    sqrtPriceMinInQ72: TickMath.getSqrtPriceAtTick(minTick),
    sqrtPriceMaxInQ72: TickMath.getSqrtPriceAtTick(maxTick),
    // activeLiquidityScalerInQ72 REMOVED (always 1)
    activeLiquidityAssets: allAssets[DEPOSIT_L] - allAssets[BORROW_L],
    reservesXAssets: reserveX,
    reservesYAssets: reserveY,
    hasBorrow: true
});
```

### 7. Asset Whitelist for Phase 1

```solidity
// In Callback Contract
mapping(address => bool) public whitelistedPairs;

function addWhitelistedPair(address pair) external onlyOwner {
    whitelistedPairs[pair] = true;
}

modifier onlyWhitelistedPair() {
    require(whitelistedPairs[ammalgamPairAddress], "Pair not whitelisted");
    _;
}
```

### 8. Support for Partial Liquidations

**Key Concept:**
- Large positions may qualify for partial liquidations over multiple tranches
- 25 ticks = 1 tranche
- LTV of entire position ≠ LTV of partial position

**Integration with Validation Library:**
```solidity
// Use updated utility with tranches = 1
bool isLiquidatable = Validation.validateIsLiquidatable(
    userAssets,
    sqrtPriceMin,
    sqrtPriceMax,
    activeLiquidity
);

if (isLiquidatable) {
    // Calculate partial liquidation if needed
    // Use LiquidationUtils or Validation library
}
```

---

## System Flow

### Phase 1: Contract Deployment

**Step 1: Deploy Callback Contract**
```
User → Deploy Callback Contract (Sepolia/Origin Chain)

Constructor Parameters:
- ammalgamPairAddress: Address of Ammalgam pair contract
- saturationContractAddress: Address of Saturation singleton
- tokenControllerAddress: Address of TokenController (for totalAssetsAndShares)
```

**Step 2: Deploy Reactive Contract**
```
User → Deploy Reactive Contract (Reactive Network)

Constructor Parameters:
- callbackContractAddress: Address of Callback Contract (Origin)
- ammalgamPairAddress: Address of Ammalgam Pair (Origin)

Automatic Subscriptions:
✓ Subscribe to ALL required events from Callback Contract
✓ Subscribe to Ammalgam's Swap event
✓ Subscribe to Ammalgam's Liquidate event (new format)
```

---

### Phase 2: User Subscribes to Protection

**User calls `subscribeProtection(cronInterval, tickMovementThreshold)` on Callback Contract**

**Parameters:**
- `cronInterval`: Time between checks (12 minutes, 2hrs or 28 hours)
- `tickMovementThreshold`: Tick movement to trigger check (e.g., 5 ticks)

**Process:**

1. **Callback Contract reads Ammalgam state:**
   ```solidity
   // Get reserves
   (uint112 reserveX, uint112 reserveY, uint32 timestamp) = 
       ITokenController(tokenController).getReserves();
   
   // Get total assets AND shares with interest (NEW!)
   (uint112[6] memory allAssets, uint112[6] memory allShares) = 
       ITokenController(tokenController).totalAssetsAndShares(true);
   ```

2. **Get user's position:**
   ```solidity
   // For each of 6 tokens, get user's shares and convert to assets
   uint256[6] memory userAssets;
   for (uint i = 0; i < 6; i++) {
       uint112 userShares = IAmmalgamERC20(tokens[i]).balanceOf(user);
       
       // Convert shares to assets using totalAssetsAndShares data
       userAssets[i] = (uint256(userShares) * allAssets[i]) / allShares[i];
   }
   ```

3. **Calculate current tick from reserves (NEW!):**
   ```solidity
   // Use TickMath helper
   int16 currentTick = TickMath.getTickFromReserves(reserveX, reserveY);
   ```

4. **Get tick range from Saturation contract (NEW!):**
   ```solidity
   ISaturationAndGeometricTWAPState saturationState = 
       ISaturationAndGeometricTWAPState(saturationContractAddress);
   
   (int16 minTick, int16 maxTick) = saturationState.getTickRange(
       address(ammalgamPair),
       reserveX,
       reserveY,
       true  // includeLongTermTick for TWAP protection
   );
   ```

5. **Build InputParams struct (UPDATED!):**
   ```solidity
   Validation.InputParams memory params = Validation.InputParams({
       userAssets: userAssets,  // [6] array
       sqrtPriceMinInQ72: TickMath.getSqrtPriceAtTick(minTick),
       sqrtPriceMaxInQ72: TickMath.getSqrtPriceAtTick(maxTick),
       // activeLiquidityScalerInQ72 REMOVED!
       activeLiquidityAssets: allAssets[DEPOSIT_L] - allAssets[BORROW_L],
       reservesXAssets: reserveX,
       reservesYAssets: reserveY,
       hasBorrow: true
   });
   ```

6. **Calculate health metrics:**
   ```solidity
   // Use Validation library for standardized calculations
   Validation.CheckLtvParams memory checkLtvParams = 
       Validation.getCheckLtvParams(
           userAssets,
           params.sqrtPriceMinInQ72,
           params.sqrtPriceMaxInQ72
       );
   
   (uint256 netDebtInL, uint256 netCollateralInL, bool netDebtX) = 
       Validation.calcDebtAndCollateral(checkLtvParams);
   
   // Health Factor = (collateral * LTVMAX) / debt
   // LTVMAX = 9000 (90% in basis points)
   uint256 healthFactor = netCollateralInL > 0 
       ? (netCollateralInL * 9000) / netDebtInL 
       : type(uint256).max;
   
   // LTV = debt / collateral (in bips)
   uint256 currentLTV = netCollateralInL > 0
       ? (netDebtInL * 10000) / netCollateralInL 
       : 0;
   ```

7. **Store position in Callback Contract:**
   ```solidity
   positions[user] = Position({
       isActive: true,
       cronInterval: cronInterval,
       tickMovementThreshold: tickMovementThreshold,  // NEW - in ticks
       lastTick: currentTick,                          // NEW - baseline tick
       lastHealthFactor: healthFactor,
       thresholdHealthFactor: 1.2e18,                 // Default threshold (1.2x)
       lastCheckTimestamp: block.timestamp
   });
   ```

8. **Emit PositionSubscribed event:**
   ```solidity
   event PositionSubscribed(
       address indexed user,
       uint256 cronInterval,
       uint256 tickMovementThreshold,  // NEW - in ticks
       int16 currentTick,               // NEW - current tick
       uint256 healthFactor,
       uint256 currentLTV,
       uint256 thresholdHealthFactor,
       uint256 collateralInL,
       uint256 debtInL,
       uint256 timestamp
   );
   ```

---

### Phase 3A: Real-Time Tick Monitoring (UPDATED)

**Reactive Contract monitors Ammalgam Swap events continuously**

**Flow:**

1. **Ammalgam emits Swap event:**
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

2. **Reactive Contract detects Swap event:**
   ```
   Reactive Contract receives Swap event
   → Triggered on EVERY swap transaction
   ```

3. **Reactive Contract sends callback for tick update:**
   ```
   Reactive Contract → Callback Contract: emitTickDetails()
   
   Purpose: Read current tick from reserves
   ```

4. **Callback Contract reads current state:**
   ```solidity
   // Get current reserves
   (uint112 reserveX, uint112 reserveY,) = 
       ITokenController(tokenController).getReserves();
   
   // Calculate current tick
   int16 currentTick = TickMath.getTickFromReserves(reserveX, reserveY);
   ```

5. **Callback Contract emits TickUpdated event (NEW!):**
   ```solidity
   event TickUpdated(
       int16 currentTick,      // Current tick from reserves
       uint112 reserveX,       // For reference
       uint112 reserveY,       // For reference
       uint256 timestamp
   );
   ```

6. **Reactive Contract receives TickUpdated event:**
   ```
   - Decodes: currentTick, reserveX, reserveY
   - For each monitored user:
       - Compare to baseline: userBaselineTick[user]
       - Calculate tick movement
   ```

7. **Reactive Contract checks tick movement threshold:**
   ```javascript
   For each user with active protection:
   
   tickMovement = abs(currentTick - baselineTick);
   
   if (tickMovement >= user.tickMovementThreshold) {
       → Tick threshold exceeded!
       → Send callback: checkPosition(user)
   }
   ```

8. **Position check triggered by tick movement:**
   ```
   If tick movement threshold exceeded:
       Reactive Contract → Callback Contract: checkPosition(user)
       → Triggers protection flow
       → Updates baseline tick after check
   ```

---

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
   // Get reserves
   (uint112 reserveX, uint112 reserveY,) = 
       ITokenController(tokenController).getReserves();
   
   // Get total assets and shares with interest
   (uint112[6] memory allAssets, uint112[6] memory allShares) = 
       ITokenController(tokenController).totalAssetsAndShares(true);
   
   // Get user balances
   for (uint i = 0; i < 6; i++) {
       userShares[i] = tokens[i].balanceOf(user);
       userAssets[i] = (userShares[i] * allAssets[i]) / allShares[i];
   }
   
   // Calculate current tick
   int16 currentTick = TickMath.getTickFromReserves(reserveX, reserveY);
   
   // Get tick range from Saturation contract
   (int16 minTick, int16 maxTick) = saturationState.getTickRange(
       address(ammalgamPair), reserveX, reserveY, true
   );
   
   // Build params and calculate health
   // ... (same as subscription)
   ```

4. **Emit PositionChecked event:**
   ```solidity
   event PositionChecked(
       address indexed user,
       int16 currentTick,               // NEW
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
   - Decodes: user, currentTick, healthFactor, thresholdHealthFactor
   - Updates baseline tick: userBaselineTick[user] = currentTick
   - Checks: if (healthFactor < thresholdHealthFactor)
   ```

6. **Protection execution if threshold met:**

   **If healthFactor < threshold:**
   ```
   Reactive Contract → Callback Contract: executeProtection(user)
   ```

   **Callback Contract executes protection:**
   ```solidity
   // Read latest state atomically (same as checkPosition)
   
   // Validate if position is liquidatable
   bool isLiquidatable = Validation.validateIsLiquidatable(
       params.userAssets,
       params.sqrtPriceMinInQ72,
       params.sqrtPriceMaxInQ72,
       params.activeLiquidityAssets
   );
   
   if (isLiquidatable) {
       // Calculate partial repayment needed
       uint256 targetHealthFactor = 1.5e18;  // Target 1.5x
       uint256 repayAmount = calculateRepayAmount(
           netDebtInL, 
           netCollateralInL, 
           targetHealthFactor
       );
       
       // Execute repayment on behalf of user
       // User must have pre-approved tokens
       (uint256 repayX, uint256 repayY) = ammalgamPair.repay(user);
       
       // Ammalgam internally validates with validateOnUpdate()
   }
   ```

   **Emit ProtectionExecuted event:**
   ```solidity
   event ProtectionExecuted(
       address indexed user,
       uint256 repaidXAssets,       // NEW - separate X and Y
       uint256 repaidYAssets,       // NEW
       uint256 oldHealthFactor,
       uint256 newHealthFactor,
       uint256 gasUsed,
       uint256 timestamp
   );
   ```

---

### Phase 4: Ammalgam Liquidate Event Monitoring (UPDATED)

**Reactive Contract is subscribed to Ammalgam's Liquidate event**

**When liquidation occurs:**

1. **Ammalgam emits Liquidate event (NEW format!):**
   ```solidity
   event Liquidate(
       address indexed borrower,
       address indexed to,              // Liquidator
       uint256 seizedLAssets,           // Seized from hard deposit
       uint256 seizedXAssets,
       uint256 seizedYAssets,
       uint256 repayXAssets,            // Credit requested
       uint256 repayYAssets,
       uint256 actualRepaidXAssets,     // Actually repaid
       uint256 actualRepaidYAssets,
       uint256 liquidationType          // HARD=0, SATURATION=1, LEVERAGE=2
   );
   ```

2. **Reactive Contract detects event:**
   ```
   - Decodes: borrower, liquidationType, amounts
   - Determines if borrower is a monitored user
   - Checks liquidation type (HARD, SATURATION, or LEVERAGE)
   ```

3. **Send callback to verify position:**
   ```
   Reactive Contract → Callback Contract: checkPosition(borrower)
   ```

4. **Callback Contract checks post-liquidation state:**
   ```
   - Reads updated position from Ammalgam
   - Calculates remaining health factor
   - Determines if position still at risk
   ```

5. **Reactive Contract evaluates:**
   ```
   If position still at risk AND liquidationType == HARD:
       → Consider protection for remaining position
       → May execute additional protective repayment
   
   If position closed or healthy:
       → Monitoring continues or subscription ends
   ```

---

## Data Access & Calculations

### External Functions Used (UPDATED)

1. **`ITokenController.totalAssetsAndShares(bool withInterest)` (NEW!)**
   - Returns: `(uint112[6] memory allAssets, uint112[6] memory allShares)`
   - Replaces: separate calls to `totalAssets()` and `totalShares()`
   - Benefits: Single call, includes interest, includes protocol fees

2. **`ITokenController.getReserves()`**
   - Returns: `(uint112 reserveXAssets, uint112 reserveYAssets, uint32 lastTimestamp)`
   - Used: For tick calculation and price monitoring

3. **`ISaturationAndGeometricTWAPState.getTickRange()` (NEW!)**
   - Parameters: `(address pair, uint256 reserveX, uint256 reserveY, bool includeLongTermTick)`
   - Returns: `(int16 minTick, int16 maxTick)`
   - Used: For LTV calculations with TWAP-based tick bounds

4. **`TickMath.getTickFromReserves()` (NEW!)**
   - Parameters: `(uint256 reserveXAssets, uint256 reserveYAssets)`
   - Returns: `int16 currentTick`
   - Used: Convert reserves to tick for monitoring

5. **Token `balanceOf(user)`**
   - Called on each of 6 token contracts
   - Returns: user's share balance

6. **`IAmmalgamPair.liquidate()` (UPDATED signature!)**
   - Parameters: 
     ```solidity
     (
         address borrower,
         address to,
         uint256 seizedLAssets,
         uint256 seizedXAssets,
         uint256 seizedYAssets,
         uint256 repayXAssets,
         uint256 repayYAssets,
         uint256 liquidationType  // 0=HARD, 1=SATURATION, 2=LEVERAGE
     )
     ```

### Tick-Based Calculations (NEW)

**Tick Movement Calculation:**
```solidity
int16 tickMovement = currentTick > baselineTick 
    ? currentTick - baselineTick 
    : baselineTick - currentTick;

bool thresholdExceeded = tickMovement >= tickMovementThreshold;
```

**Converting Tick to Price:**
```solidity
// For reference/logging only
uint256 sqrtPriceQ72 = TickMath.getSqrtPriceAtTick(currentTick);
uint256 priceQ128 = (sqrtPriceQ72 * sqrtPriceQ72) >> 16;
```

**Position-Dependent Thresholds:**
```solidity
// Critical positions (HF 1.0-1.15): tickThreshold = 1
// High risk (HF 1.15-1.3): tickThreshold = 2-3
// Medium risk (HF 1.3-1.5): tickThreshold = 5
// Low risk (HF 1.5-2.0): tickThreshold = 10
// Very safe (HF > 2.0): tickThreshold = 25 (1 tranche)
```

---

## Events Reference

### Callback Contract Events

#### PositionSubscribed (UPDATED)
```solidity
event PositionSubscribed(
    address indexed user,
    uint256 cronInterval,
    uint256 tickMovementThreshold,  // In ticks (not bips!)
    int16 currentTick,               // Current tick (not price!)
    uint256 healthFactor,
    uint256 currentLTV,
    uint256 thresholdHealthFactor,
    uint256 collateralInL,
    uint256 debtInL,
    uint256 timestamp
);
```

#### PositionChecked (UPDATED)
```solidity
event PositionChecked(
    address indexed user,
    int16 currentTick,               // In ticks
    uint256 healthFactor,
    uint256 currentLTV,
    uint256 thresholdHealthFactor,
    uint256 collateralInL,
    uint256 debtInL,
    uint256 timestamp
);
```

#### TickUpdated (NEW!)
```solidity
event TickUpdated(
    int16 currentTick,
    uint112 reserveX,
    uint112 reserveY,
    uint256 timestamp
);
```

#### ProtectionExecuted (UPDATED)
```solidity
event ProtectionExecuted(
    address indexed user,
    uint256 repaidXAssets,           // Separated by asset
    uint256 repaidYAssets,
    uint256 oldHealthFactor,
    uint256 newHealthFactor,
    uint256 gasUsed,
    uint256 timestamp
);
```

### Ammalgam Events

#### Swap (unchanged)
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

#### Liquidate (UPDATED!)
```solidity
event Liquidate(
    address indexed borrower,
    address indexed to,
    uint256 seizedLAssets,           // From deposits
    uint256 seizedXAssets,
    uint256 seizedYAssets,
    uint256 repayXAssets,            // Credit requested
    uint256 repayYAssets,
    uint256 actualRepaidXAssets,     // Actually repaid
    uint256 actualRepaidYAssets,
    uint256 liquidationType          // 0=HARD, 1=SATURATION, 2=LEVERAGE
);
```

---


### Demo Tick-Based Configurations

```
Position Risk Level → Configuration

CRITICAL (HF 1.0-1.15):
├─ Cron: Every 12 seconds
├─ Tick Threshold: 1 tick
└─ Max Protection: Fastest response, highest cost

HIGH RISK (HF 1.15-1.3):
├─ Cron: Every 1 minute
├─ Tick Threshold: 2-3 ticks
└─ Strong Protection: Fast response, high cost

MEDIUM RISK (HF 1.3-1.5):
├─ Cron: Every 12 minutes
├─ Tick Threshold: 5 ticks
└─ Balanced: Good protection, moderate cost

LOW RISK (HF 1.5-2.0):
├─ Cron: Every 2 hours
├─ Tick Threshold: 10 ticks
└─ Cost Efficient: Basic protection, low cost

VERY SAFE (HF > 2.0):
├─ Cron: Every 28 hours
├─ Tick Threshold: 25 ticks (1 tranche)
└─ Minimum Cost: Emergency protection only
```
