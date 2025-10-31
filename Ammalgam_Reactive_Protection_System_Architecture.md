```mermaid
sequenceDiagram
    participant User
    participant CallbackContract as Callback Contract<br/>(Sepolia)
    participant AmmalgamPair as Ammalgam Protocol<br/>(Sepolia)
    participant Events as Event Logs<br/>(Sepolia Chain)
    participant ReactiveContract as Reactive Smart Contract<br/>(Lasna Testnet)
    
    Note over User,ReactiveContract: PHASE 1: CONTRACT DEPLOYMENT
    
    User->>CallbackContract: 1. Deploy Callback Contract
    Note over CallbackContract: Constructor params:<br/>- ammalgamPairAddress<br/>- tokenAddresses[6]
    
    User->>ReactiveContract: 2. Deploy Reactive Contract<br/>Subscribe to Callback Contract events
    Note over ReactiveContract: Constructor params:<br/>- callbackContractAddress (Sepolia)<br/>Subscribe to ALL events from Callback<br/>Subscribe to Ammalgam Liquidate event (Topic0)
    
    Note over User,ReactiveContract: PHASE 2: USER SUBSCRIBES TO PROTECTION
    
    User->>CallbackContract: 3. subscribeProtection(cronInterval)
    Note over User: cronInterval: 12sec to 28hours
    
    CallbackContract->>AmmalgamPair: getReserves()
    AmmalgamPair-->>CallbackContract: (reserveX, reserveY, timestamp)
    
    CallbackContract->>AmmalgamPair: totalAssets()
    AmmalgamPair-->>CallbackContract: [depositL, depositX, depositY,<br/>borrowL, borrowX, borrowY]
    
    CallbackContract->>AmmalgamPair: balanceOf(user) for each token[6]
    AmmalgamPair-->>CallbackContract: [userShares[6]]
    
    CallbackContract->>CallbackContract: Convert shares to assets<br/>userAssets[i] = userShares[i] * totalAssets[i] / totalShares[i]
    
    CallbackContract->>CallbackContract: Calculate tick from reserves<br/>tick = TickMath.getTickAtPrice(reserveX * Q128 / reserveY)
    
    CallbackContract->>AmmalgamPair: getTickRange()
    AmmalgamPair-->>CallbackContract: (minTick, maxTick)
    
    CallbackContract->>CallbackContract: Build InputParams:<br/>- userAssets[6]<br/>- sqrtPriceMin/Max from ticks<br/>- activeLiquidityScaler<br/>- activeLiquidityAssets<br/>- reserves
    
    CallbackContract->>CallbackContract: Calculate Health Metrics:<br/>- convertXToL, convertYToL<br/>- netDepositedXinL, netDepositedYinL<br/>- netBorrowedXinL, netBorrowedYinL<br/>- healthFactor = collateral * LTVMAX / debt<br/>- currentLTV = debt / collateral
    
    CallbackContract->>CallbackContract: Store Position:<br/>- user address<br/>- cronInterval<br/>- lastHealthFactor<br/>- thresholdHealthFactor (e.g., 1.2e18)
    
    CallbackContract->>Events: Emit PositionSubscribed Event
    Note over Events: PositionSubscribed:<br/>- address indexed user<br/>- uint256 cronInterval<br/>- uint256 healthFactor<br/>- uint256 currentLTV<br/>- uint256 thresholdHealthFactor<br/>- uint256 collateralInL<br/>- uint256 debtInL<br/>- uint256 timestamp
    
    Events-->>ReactiveContract: Event Received: PositionSubscribed
    
    ReactiveContract->>ReactiveContract: Decode event data<br/>(user, cronInterval, healthFactor, threshold)
    
    ReactiveContract->>ReactiveContract: Subscribe user to cron:<br/>cronSchedule[cronInterval].add(user)
    
    ReactiveContract->>ReactiveContract: Check threshold:<br/>if (healthFactor < thresholdHealthFactor)
    
    alt Health Factor Below Threshold
        ReactiveContract->>CallbackContract: Immediate Callback: executeProtection(user)
        Note over ReactiveContract: Protection needed immediately
    else Health Factor Above Threshold
        Note over ReactiveContract: User subscribed to cron<br/>Wait for next cron event
    end
    
    Note over User,ReactiveContract: PHASE 3: CONTINUOUS CRON MONITORING
    
    loop Every Cron Interval (user-specific)
        Note over ReactiveContract: Cron Event Triggered<br/>for cronInterval X
        
        ReactiveContract->>ReactiveContract: Get users for this interval:<br/>users = cronSchedule[cronInterval]
        
        loop For Each User in Cron Schedule
            ReactiveContract->>CallbackContract: Callback: checkPosition(user)
            
            CallbackContract->>AmmalgamPair: getReserves()
            AmmalgamPair-->>CallbackContract: (reserveX, reserveY, timestamp)
            
            CallbackContract->>AmmalgamPair: totalAssets()
            AmmalgamPair-->>CallbackContract: [depositL, depositX, depositY,<br/>borrowL, borrowX, borrowY]
            
            CallbackContract->>AmmalgamPair: balanceOf(user) for each token[6]
            AmmalgamPair-->>CallbackContract: [userShares[6]]
            
            CallbackContract->>CallbackContract: Convert shares to assets
            
            CallbackContract->>CallbackContract: Calculate current tick
            
            CallbackContract->>AmmalgamPair: getTickRange()
            AmmalgamPair-->>CallbackContract: (minTick, maxTick)
            
            CallbackContract->>CallbackContract: Build InputParams
            
            CallbackContract->>CallbackContract: Calculate Health Metrics:<br/>- healthFactor<br/>- currentLTV<br/>- collateralInL<br/>- debtInL
            
            CallbackContract->>Events: Emit PositionChecked Event
            Note over Events: PositionChecked:<br/>- address indexed user<br/>- uint256 healthFactor<br/>- uint256 currentLTV<br/>- uint256 thresholdHealthFactor<br/>- uint256 collateralInL<br/>- uint256 debtInL<br/>- uint256 timestamp
            
            Events-->>ReactiveContract: Event Received: PositionChecked
            
            ReactiveContract->>ReactiveContract: Decode event:<br/>(user, healthFactor, threshold)
            
            ReactiveContract->>ReactiveContract: Check condition:<br/>if (healthFactor < thresholdHealthFactor)
            
            alt Health Factor Below Threshold - PROTECTION NEEDED
                ReactiveContract->>CallbackContract: Callback: executeProtection(user)
                
                CallbackContract->>AmmalgamPair: Read latest state<br/>(getReserves, totalAssets, balanceOf)
                AmmalgamPair-->>CallbackContract: Current state
                
                CallbackContract->>CallbackContract: Recalculate metrics atomically
                
                CallbackContract->>CallbackContract: Calculate protection amount:<br/>targetHF = 1.5e18<br/>repayAmount = currentDebt - (collateral * LTVMAX / targetHF)
                
                CallbackContract->>AmmalgamPair: Execute protection<br/>(e.g., repay debt on behalf of user)
                Note over CallbackContract: User must have pre-approved<br/>tokens or provided funds
                
                AmmalgamPair->>AmmalgamPair: Process repayment<br/>validateOnUpdate(user, user, true)<br/>Update saturation
                AmmalgamPair-->>CallbackContract: Success
                
                CallbackContract->>CallbackContract: Update stored position data
                
                CallbackContract->>Events: Emit ProtectionExecuted Event
                Note over Events: ProtectionExecuted:<br/>- address indexed user<br/>- uint256 repaidAmount<br/>- uint256 repaidAssetType (L/X/Y)<br/>- uint256 oldHealthFactor<br/>- uint256 newHealthFactor<br/>- uint256 gasUsed<br/>- uint256 timestamp
                
                Events-->>ReactiveContract: Event Received: ProtectionExecuted
                Note over ReactiveContract: Stateless processing<br/>Log success, continue monitoring
                
            else Health Factor Above Threshold - HEALTHY
                Note over ReactiveContract: Position healthy<br/>Continue monitoring
            end
        end
    end
    
    Note over User,ReactiveContract: PHASE 4: AMMALGAM LIQUIDATE EVENT MONITORING
    
    AmmalgamPair->>Events: Ammalgam emits: Liquidate Event
    Note over Events: Liquidate (from Ammalgam):<br/>- address indexed borrower<br/>- address indexed to<br/>- uint256 depositL<br/>- uint256 depositX<br/>- uint256 depositY<br/>- uint256 repayLX<br/>- uint256 repayLY<br/>- uint256 repayX<br/>- uint256 repayY<br/>- uint256 liquidationType
    
    Events-->>ReactiveContract: Liquidate event detected<br/>(subscribed to Topic0)
    
    ReactiveContract->>ReactiveContract: Decode liquidation event<br/>(borrower, liquidationType, amounts)
    
    ReactiveContract->>CallbackContract: Callback: checkPosition(borrower)
    Note over ReactiveContract: Check if borrower is a monitored user<br/>Verify position status post-liquidation
    
    CallbackContract->>AmmalgamPair: Read position state
    AmmalgamPair-->>CallbackContract: Updated position data
    
    CallbackContract->>CallbackContract: Calculate post-liquidation health
    
    CallbackContract->>Events: Emit PositionChecked Event
    
    Events-->>ReactiveContract: Position status after liquidation
    
    alt Position Still At Risk
        ReactiveContract->>CallbackContract: executeProtection(borrower)
        Note over ReactiveContract: Attempt to save remaining position
    else Position Closed or Healthy
        Note over ReactiveContract: Monitoring continues or ends
    end
    
    Note over User,ReactiveContract: SUMMARY - KEY FUNCTION CALLS
    
    Note over CallbackContract: Callback Contract Functions:<br/>1. subscribeProtection(cronInterval)<br/>2. checkPosition(user)<br/>3. executeProtection(user)
    
    Note over Events: Emitted Events:<br/>1. PositionSubscribed<br/>2. PositionChecked<br/>3. ProtectionExecuted
    
    Note over ReactiveContract: Reactive Contract:<br/>- Subscribes to callback events<br/>- Subscribes to Ammalgam Liquidate (Topic0)<br/>- Manages cron scheduling<br/>- Sends callbacks based on events<br/>- STATELESS (no storage) ```
