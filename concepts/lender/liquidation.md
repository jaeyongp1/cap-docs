# Liquidation

When an Operator's health factor drops below 1, liquidators can purchase the Operator's delegated collateral by repaying their outstanding debt.&#x20;

## Mechanics

The liquidation process is as follows:

1. The liquidation process kicks off with a liquidator calling the <kbd>initiateLiquidation</kbd> function&#x20;
2. After the liquidation has been initiated, the Operator has a grace period of 12 hours to improve the health of the loan
   1. If the current Loan-to-Value is less than the emergency liquidation threshold, the grace period is overridden, opening the liquidation window immediately.&#x20;
3. At the end of the grace period, liquidators can call the <kbd>liquidate</kbd> function until the expiry of the liquidation period. Liquidations will expire in 3 days after the end of the grace period.&#x20;
4. A successful liquidation will execute a repayment of the borrowed asset, reducing the debt by the liquidation amount
5. The liquidated amount and the liquidation bonus is slashed from the Shared Security Network, transferring the collateral to the liquidator
6. The liquidation window will close once the Operator's health is recovered, i.e. over 1

#### Liquidation Threshold

By default, the liquidation threshold is set to 80% LTV. If the current LTV exceeds the liquidation threshold, then a liquidation may be opened. An emergency liquidation mechanism is used to override the grace period if the health threshold drops significantly. The emergency liquidation threshold is set to 90% LTV.&#x20;

#### Liquidation Expiry

All liquidations have an expiry window. The window ensures that once the position becomes healthy again, the Operator will no longer be liquidatable. Hence, if the Operator were to be liquidatable again, the Operator will be ensured another grace period. If the Operator's health factor is below 1 after the expiry, any liquidator can initiate the liquidation process again.&#x20;

#### Target Health and Maximum Liquidatable Amount

Target health is a health ratio threshold that defines the desired health level that liquidations should restore an Operator's position to, currently set at 1.25. It acts as a "safety buffer" that liquidations aim to achieve.

The target health defines the the maximum liquidatable amount. The maximum liquidatable amount is calculated as:&#x20;

<kbd>((Target Health \* Total Debt) - (Total Delegation \* Liquidation Threshold)) / ((Target Health - Liquidation Threshold) \* Asset Price)</kbd>

In other words, the goal is for the health factor, i.e.&#x20;

<kbd>Total Delegation \* Liquidation Threshold / (TotalDebt - Liquidated Amount)</kbd>

to be equal to the target health ratio.

#### Liquidation Bonus

The protocol implements a dynamic liquidation bonus system that provides incentives for liquidators. The liquidation bonus rises linearly with time from the grace period until the maximum amount reaches the bonus cap.  At expiry, liquidators earn the maximum bonus which is set to 10%. If an emergency liquidation is triggered, liquidators bypass the grace period, immediately earning the maximum bonus.&#x20;

Note that the bonus is only available when the total delegation exceeds total debt. The protocol prevents over-liquidating beyond available collateral

## Architecture

### Liquidation Logic

The **LiquidationLogic** library is the core component responsible for managing liquidation operations

### Key Functions

**`openLiquidation`**: Opens a liquidation window for an unhealthy operator

```solidity
function openLiquidation(ILender.LenderStorage storage $, address _agent) external
```

**Process Flow**:

1.  **Health Check**:

    ```solidity
    (,,,,, uint256 health) = ViewLogic.agent($, _agent);
    ```
2.  **Validation**:

    ```solidity
    ValidationLogic.validateOpenLiquidation(health, $.liquidationStart[_agent], $.expiry);
    ```
3.  **Window Opening**:

    ```solidity
    $.liquidationStart[_agent] = block.timestamp;
    ```

**Validation Rules**:

* Agent health must be below 1e27 (100%)
* Previous liquidation window must have expired
* Cannot open multiple concurrent windows



**`closeLiquidation`**: Closes a liquidation window when an Operator becomes healthy

```solidity
function closeLiquidation(ILender.LenderStorage storage $, address _agent) external
```

**Process Flow**:

1.  **Health Verification**:

    ```solidity
    (,,,,, uint256 health) = ViewLogic.agent($, _agent);
    ```
2.  **Validation**:

    ```solidity
    ValidationLogic.validateCloseLiquidation(health);
    ```
3.  **Window Closure**:

    ```solidity
    _closeLiquidation($, _agent);
    ```

**Validation Rules**:

* Agent health must be above 1e27 (100%)
* Only healthy agents can have windows closed



**`liquidate`**: Executes liquidation of unhealthy positions with bonus incentives

```solidity
function liquidate(ILender.LenderStorage storage $, ILender.RepayParams memory params, uint256 _minLiquidatedValue)
    external
    returns (uint256 liquidatedValue)
```

**Process Flow**:

1. An Operator must be eligible to be liquidated: health factor below 1, and have outstanding debt to be liquidated.
2. Checks for emergency liquidations, and whether liquidate timestamp is within the liquidation window.

```solidity
ValidationLogic.validateLiquidation(
    health,
    totalDelegation * $.emergencyLiquidationThreshold / totalDebt,
    $.liquidationStart[params.agent],
    $.grace,
    $.expiry
);
```

3.  Asset Pricing and Bonus Calculation

    ```solidity
    (uint256 assetPrice,) = IOracle($.oracle).getPrice(params.asset);
    uint256 bonus = ViewLogic.bonus($, params.agent);
    ```
4.  Calculate Max Liquidatable Amount

    ```solidity
    uint256 maxLiquidation = ViewLogic.maxLiquidatable($, params.agent, params.asset);
    uint256 liquidated = params.amount > maxLiquidation ? maxLiquidation : params.amount;
    ```
5.  Repay Debt:

    ```solidity
    liquidated = BorrowLogic.repay(
        $,
        ILender.RepayParams({
            agent: params.agent,
            asset: params.asset,
            amount: liquidated,
            caller: params.caller
        })
    );
    ```
6.  Liquidation Value Calculation

    ```solidity
    liquidatedValue = (liquidated + (liquidated * bonus / 1e27)) * assetPrice / (10 ** $.reservesData[params.asset].decimals);
    if (totalSlashableCollateral < liquidatedValue) liquidatedValue = totalSlashableCollateral;
    ```
7.  Execute Slashing

    ```solidity
    if (liquidatedValue > 0) IDelegation($.delegation).slash(params.agent, params.caller, liquidatedValue);
    ```
8.  Close Window

    ```solidity
    if (health >= 1e27) _closeLiquidation($, params.agent);
    ```

#### Admin Functions

**`setGrace`**: Set grace period for liquidations

```solidity
function setGrace(uint256 _grace) external;
```

**`setExpiry`**: Set expiry period for liquidations

```solidity
function setExpiry(uint256 _expiry) external;
```

**Set Bonus Cap for liquidations**

```solidity
function setBonusCap(uint256 _bonusCap) external;
```

#### View Functions

**`grace`**: View grace period in seconds&#x20;

```solidity
function grace() external view returns (uint256 gracePeriod)
```

**`expiry`**: Returns the expiry period after which liquidation rights expire

```solidity
function expiry() external view returns (uint256 expiryPeriod)
```

**`maxLiquidatable`**: Max liquidatable amount for an agent borrowing a particular asset

```solidity
function maxLiquidatable(address _agent, address _asset) external view returns (uint256 maxLiquidatableAmount)
```

**`bonus`**: Calculates the maximum bonus percentage for liquidating an agent

```solidity
function bonus(address _agent) external view returns (uint256 maxBonus)
```

**`bonusCap`**: Returns the maximum bonus percentage for liquidations

```solidity
function bonusCap() external view returns (uint256 cap)
```

**`emergencyLiquidationThreshold`**: Returns the health threshold below which grace periods are ignored

```solidity
function emergencyLiquidationThreshold() external view returns (uint256 threshold)
```

**`liquidationStart`**: Returns the timestamp when liquidation was initiated for an agent

```solidity
function liquidationStart(address _agent) external view returns (uint256 startTime)
```
