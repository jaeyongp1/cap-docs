# Lender

The Lending Pool module manages the borrowing, repayment, and liquidation processes for Operators.&#x20;

## Overview of Operations

1. [**Borrow/Repay**](borrow.md): Operators can borrow reserve assets against their delegated collateral
2. [**Liquidation**](liquidation.md): Multi-stage liquidation with grace periods and bonus incentives
3. [**Interest Rate Calculation**](interest-rates.md)**:** Dynamic protocol rates and fixed Operator specific rates

## Key Loan Parameters

* **Total Delegation:** Total amount of an Operator's received collateral from Delegators&#x20;
* **Total Slashable Collateral:** Total amount of collateral that can be slashed from a Delegator
* **Total Debt:** An Operator's total debt denominated in USD. Interest accrues to Total Debt
* **Initial Loan-to-Value (LTV):** The maximum amount an Operator can borrow relative to the Total Slashable Collateral. A 50% LTV implies that delegations must be at least 2x the size of the borrow.
* **Current LTV**: An Operator's current Loan-to-Value ratio, calculated as&#x20;
  * (Total Debt / Total Delegation) \* 100%
* **Liquidation Threshold**: Threshold that determines when an Operator's position becomes slashable in LTV ratio. By default, the threshold is set to 80%.
* **Health Factor**: Represents an Operator's loan health
  * Calculated as (Total Delegation \* Liquidation Threshold) / Total Debt
  * Liquidations are triggered when Health is below 1
* **Grace Period**: Period for Operators to recover the health before liquidation can occur. Set to 12 hours
* **Expiry Period:** Period after which liquidation rights expire. Set to 3 days
* **Target health:** Target health ratio that liquidators aim to achieve when slashing an Operator. Set to 125%
* **Bonus Cap:** Maximum bonus for liquidators, set to 10%.

## Debt Management

The protocol has two distinct types of interest:

1. **Vault Interest**: Interest paid to the Vault (stcUSD holders)
2. **Restaker Interest**: Interest paid to Delegators who provide collateral coverage

Debt in Cap is managed via Debt tokens. Debt tokens are non-transferrable ERC-20 tokens that track an Operator's debt. Debt tokens are minted when an Operator borrows, and burned when the debt is repayed.&#x20;

Interest automatically accrues to the debt token per asset, where the interest rate is calculated based on the [interest rate mechanism](interest-rates.md). Accrued interest is handled automatically via index-based scaling, inherited from the ScaledToken base class. &#x20;

## Architecture

The Lending Pool module is designed as an upgradeable proxy with modular logic libraries for maintainability and security.

1. **BorrowLogic Library** - Core logic for borrowing, repaying, and interest management
2. **LiquidationLogic Library** - Advanced liquidation mechanisms and bonus calculations
3. **ReserveLogic Library** - Reserve management and configuration
4. **ViewLogic Library** - View functions and calculations
5. **ValidationLogic Library** - Comprehensive validation systems

### Lender Contract

The Lender contract is the main interface for all lending operations.

#### Core Functions

#### Borrowing Operations

**`borrow`**: Allows agents to borrow assets from the lending pool via the BorrowLogic library

```solidity
function borrow(address _asset, uint256 _amount, address _receiver) external returns (uint256 borrowed)
```

* `_asset`: Asset to borrow
* `_amount`: Amount to borrow
* `_receiver`: Address to receive the borrowed assets

**`repay`**: Allows agents to repay assets from the lending pool via the BorrowLogic library

```solidity
function repay(address _asset, uint256 _amount, address _agent) external returns (uint256 repaid)
```

* `_asset`: Asset to repay
* `_amount`: Amount to repay
* `_agent`: Agent whose debt is being repaid



**Liquidation**

**`openLiquidation`**: Opens a liquidation window for an unhealthy operator

```solidity
function openLiquidation(address _agent) external
```

**`liquidate`**: Liquidate debt and slash collateral

```solidity
function liquidate(address _agent, address _asset, uint256 _amount, uint256 _minLiquidatedValue) external returns (uint256 liquidatedValue)
```

**`closeLiquidation`**: End liquidation window when position is healthy

```solidity
function closeLiquidation(address _agent) external
```

**Interest Distribution**

Convert accrued interest into debt tokens via the BorrowLogic library

```solidity
function realizeInterest(address _asset) external returns (uint256 actualRealized)
function realizeRestakerInterest(address _agent, address _asset) external returns (uint256 actualRealized)
```

**Types**:

* Vault Interest: System-wide interest realization
* Restaker Interest: Agent-specific interest for restakers

#### **Admin Functions**

**`addAsset`**: Adds new assets to the lending pool

```solidity
function addAsset(AddAssetParams calldata _params) external checkAccess(this.addAsset.selector)
```

Parameters:

* **asset**: Asset address to add
* **vault**: Vault address for the asset
* **debtToken**: Debt token address
* **interestReceiver**: Interest receiver address
* **bonusCap**: Liquidation bonus cap
* **minBorrow**: Minimum borrow amount

**`removeAsset`:** Removes assets from the lending pool, if there are no outstanding debt before removal

```solidity
function removeAsset(address _asset) external checkAccess(this.removeAsset.selector)
```

**`papuseAsset`**: Pause asset from being borrowed, prevents new borrows when paused

```solidity
function pauseAsset(address _asset, bool _pause) external checkAccess(this.pauseAsset.selector)
```

**`setInterestReceiver`**: Sets interest receiver for an asset

```solidity
function setInterestReceiver(address _asset, address _interestReceiver) external checkAccess(this.setInterestReceiver.selector)
```

**`setMinBorrow`:** Set Minimum Borrow

```solidity
function setMinBorrow(address _asset, uint256 _minBorrow) external checkAccess(this.setMinBorrow.selector)
```

**Set Liquidation Parameters**

More details can be found in the [Liquidation](liquidation.md) page

```solidity
function setGrace(uint256 _grace) external checkAccess(this.setGrace.selector)
function setExpiry(uint256 _expiry) external checkAccess(this.setExpiry.selector)
function setBonusCap(uint256 _bonusCap) external checkAccess(this.setBonusCap.selector)
function setTargetHealth(uint256 _targetHealth) external checkAccess(this.setTargetHealth.selector)
```

#### **View Functions**

**`Agent`:** View agent(Operator) information

```solidity
function agent(address _agent) external view returns (
    uint256 totalDelegation,        // Total delegation in USD (8 decimals)
    uint256 totalSlashableCollateral, // Total slashable collateral in USD (8 decimals)
    uint256 totalDebt,              // Total debt in USD (8 decimals)
    uint256 ltv,                    // Loan to value ratio (1e27)
    uint256 liquidationThreshold,   // Liquidation threshold (1e27)
    uint256 health                  // Health factor (1e27)
)
```

**`maxBorrowable`:** Maximum Borrowable Amount for an agent for an asset

```solidity
function maxBorrowable(address _agent, address _asset) external view returns (uint256 maxBorrowableAmount)
```

**`Debt`**: Get the current debt balances for an agent for a specific asset

<pre class="language-solidity"><code class="lang-solidity"><strong>function debt(address _agent, address _asset) external view returns (uint256 totalDebt);
</strong></code></pre>

### Data Structures

The LenderStorage stores the list of all the Reserve Data along with agent (Operator) configuration and liquidation parameters.&#x20;

```solidity
struct LenderStorage {
    // Addresses
    address delegation;           // Delegation contract managing agent permissions
    address oracle;               // Oracle for price feeds
    // Reserve configuration
    mapping(address => ReserveData) reservesData;  // Asset to reserve data mapping
    mapping(uint256 => address) reservesList;      // Reserve ID to asset mapping
    uint16 reservesCount;                          // Total number of reserves
    // Agent configuration
    mapping(address => AgentConfigurationMap) agentConfig;  // Agent configuration data
    mapping(address => uint256) liquidationStart;           // Liquidation start timestamps
    // Liquidation parameters
    uint256 targetHealth;                    // Target health ratio (1e27)
    uint256 grace;                           // Grace period in seconds
    uint256 expiry;                          // Liquidation expiry period
    uint256 bonusCap;                        // Maximum liquidation bonus (1e27)
    uint256 emergencyLiquidationThreshold;   // Emergency liquidation threshold
}
```

The reserve data struct stores all relevant Vault information for an underlying asset.

```solidity
struct ReserveData {
    uint256 id;                              // Reserve ID
    address vault;                           // Vault address
    address debtToken;                       // Debt token address
    address interestReceiver;                // Interest receiver address
    uint8 decimals;                          // Asset decimals
    bool paused;                             // Pause state
    uint256 debt;                            // Total debt
    uint256 totalUnrealizedInterest;         // Total unrealized interest
    mapping(address => uint256) unrealizedInterest;  // Per-agent unrealized interest
    mapping(address => uint256) lastRealizationTime; // Last interest realization time
    uint256 minBorrow;                       // Minimum borrow amount
}
```

Agent (Operator) configuration is stored in the AgentConfigurationMap struct that contains a single uint256 field called data. This field is a bitmap, where each bit represents a boolean of whether an agent is borrowing for a specific reserve. Agent specific loan information such as collateral amount, liquidation threshold is stored in the Delegation contract and can be read via the ViewLogic contract.

