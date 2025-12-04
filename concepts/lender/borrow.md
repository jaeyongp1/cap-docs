# Borrow

Whitelisted Operators in Cap can borrow and repay the underlying Vault assets, provided they have secured sufficient collateral from a Delegator.&#x20;

## Mechanics

### Borrow

The borrow process flow is as follows:

1. Operator calls the <kbd>borrow</kbd> function on the Lender contract
2. Restaker interest is realized first to prevent charging for compounded interest
3. If eligible to borrow\*, assets are lent out from the Vault to the Opeartor
4. Debt tokens are minted to track the loan&#x20;

**Key Validations\***:

* Operator must have sufficient collateral and health factor
* Operator must be whitelisted and not paused
* Asset being borrowed should not be paused
* Borrow amount must meet minimum requirements, and may not exceed maximum borrows
  * Minimum: set by Admin via the setMinBorrow function
  * Maximum: The smaller value of the Operator's remaining borrowable capacity, and the remaining available amount to be borrowed from the Vault

### Repay

The repay process flow is as follows:

1. Operator calls the <kbd>repay</kbd> function on the Lender contract
2. Restaker interest is realized first to ensure all interest is accounted for
3. The repayment is processed in the following order:
   1. Unrealized restaker interest (if any)
   2. Vault principal debt
   3. Vault interest
4. Debt tokens corresponding to the total amount repaid are burned

{% hint style="info" %}
If the repayment is not in full, the system will maintain a minimum borrow amount for the Operator.&#x20;
{% endhint %}

#### Interest Distribution

The repaid assets are distributed back to the shareholders:&#x20;

1. **Vault Principal**: Sent to [Vault](../vault/) contract's reserves
2. **Vault Interest**: Sent to `interestReceiver` (set to [FeeAuction](../fee-auction.md)). Any excess interest will go to the interest receiver
3. **Restaker Interest**: Sent to [Delegation](../delegation/) contract for distribution to Delegators
4. **Unrealized Interest**: Added to agent's debt balance for future repayment

### Realizing Interest

Interest in Cap is **accrued continuously** but **realized discretely**:

* **Accrued Interest**: Calculated in real-time based on elapsed time and rates
* **Realized Interest**: Actually borrowed from vault and distributed to recipients
* **Unrealized Interest**: Accrued but not yet realized due to vault liquidity constraints

As can be seen, interest can be realized prior to repayment in Cap. Both stcUSD holders and Delegators may permissionlessly realize interest by borrowing from the vault and distributing to interest receivers.

There are two functions to realize interest: <kbd>realizeInterest</kbd> and <kbd>realizeRestakerInterest</kbd>

RealizeInterest:

* Realizes vault interest (interest paid to the stcUSD holders)
* Interest is borrowed from the Vault and paid to the interest receiver

RealizeRestakerInterest:

* Realizes restaker interest (interest paid to Delegators)
* Interest is borrowed from the Vault and paid to the Delegation Contract

For both functions, the process flow is as follows:

1. Determine available interest to realize
2. Increase reserve debt
3. Borrow interest amount from Vault
4. Transfer assets to recipient

## Architecture

### Borrow Logic

The **BorrowLogic** library is the core component responsible for managing borrowing and repayment operations in the CAP lending system.

**Key Dependencies:**

* **IDebtToken**: Manages debt token minting/burning
* **IDelegation**: Handles restaker interest distribution
* **IVault**: Manages asset storage and transfers
* **ValidationLogic**: Validates borrowing operations
* **ViewLogic**: Calculates borrowing capacity and health metrics
* **AgentConfiguration**: Tracks agent borrowing status per reserve

#### Debt Token

Debt tokens are core to accounting an Operator's loan. Debt tokens are non-transferrable ERC20 tokens minted when an Operator borrows, and burned when the debt is repayed.&#x20;

```solidity
interface IDebtToken {
    function mint(address to, uint256 amount) external;
    function burn(address from, uint256 amount) external;
    function balanceOf(address account) external view returns (uint256);
}
```

#### Vault

Assets are transferred from/to the Vault once validation checks and interest calculations are done in the Lender.

```solidity
interface IVault {
    function borrow(address _asset, uint256 _amount, address _receiver) external;
    function repay(address _asset, uint256 _amount) external;
    function availableBalance(address _asset) external view returns (uint256);
}
```

### Key Functions

**`borrow`**: Borrow an asset

```solidity
function borrow(ILender.LenderStorage storage $, ILender.BorrowParams memory params)
    external
    returns (uint256 borrowed)
```

where the borrow params are:

```solidity
struct BorrowParams {
    address agent;        // Borrower address
    address asset;        // Asset to borrow
    uint256 amount;       // Amount to borrow
    address receiver;     // Receiver of borrowed assets
    bool maxBorrow;       // Whether to borrow maximum available
}
```

**Process Flow**:

1.  **Interest Realization**:

    ```solidity
    realizeRestakerInterest($, params.agent, params.asset);
    ```
2.  **Amount Calculation**:

    ```solidity
    if (params.maxBorrow) {
        params.amount = ViewLogic.maxBorrowable($, params.agent, params.asset);
    }
    ```
3.  **Validation**:

    ```solidity
    ValidationLogic.validateBorrow($, params);
    ```
4.  **State Updates**:

    ```solidity
    IDelegation($.delegation).setLastBorrow(params.agent);
    $.agentConfig[params.agent].setBorrowing(reserve.id, true);
    ```
5.  **Asset Transfer**:

    ```solidity
    IVault(reserve.vault).borrow(params.asset, borrowed, params.receiver);
    IDebtToken(reserve.debtToken).mint(params.agent, borrowed);
    reserve.debt += borrowed;
    ```

**`repay`**: Repay an asset

```solidity
function repay(ILender.LenderStorage storage $, ILender.RepayParams memory params)
    external
    returns (uint256 repaid)
```

where the repay params are:

```solidity
struct RepayParams {
    address agent;        // Agent to repay for
    address asset;        // Asset to repay
    uint256 amount;       // Amount to repay
    address caller;       // Address making the repayment
}
```

**Process Flow**:

1.  **Interest Realization**:

    ```solidity
    realizeRestakerInterest($, params.agent, params.asset);
    ```
2.  **Amount Calculation**: **prevents repaying more than owned**

    ```solidity
    uint256 agentDebt = IERC20(reserve.debtToken).balanceOf(params.agent);
    repaid = Math.min(params.amount, agentDebt);
    ```
3.  **Minimum Debt Protection**:

    ```solidity
    if (remainingDebt > 0 && remainingDebt < reserve.minBorrow) {
        repaid = agentDebt - reserve.minBorrow;
    }
    ```
4.  **Repayment Allocation**:

    ```solidity
    // Calculate interest repayment
    if (repaid > reserve.unrealizedInterest[params.agent] + reserve.debt) {
        interestRepaid = repaid - (reserve.debt + reserve.unrealizedInterest[params.agent]);
    }

    // Calculate restaker repayment
    if (remaining > reserve.unrealizedInterest[params.agent]) {
        restakerRepaid = reserve.unrealizedInterest[params.agent];
    }

    // Calculate vault repayment
    uint256 vaultRepaid = Math.min(remaining, reserve.debt);
    ```
5.  **Asset Distribution**:

    ```solidity
    // Send to restakers
    IERC20(params.asset).safeTransfer($.delegation, restakerRepaid);
    IDelegation($.delegation).distributeRewards(params.agent, params.asset);
    // Send to vault
    IVault(reserve.vault).repay(params.asset, vaultRepaid);
    // Send to interest receiver
    IERC20(params.asset).safeTransfer(reserve.interestReceiver, interestRepaid);
    ```
6.  **Debt Token Burning**:

    ```solidity
    IDebtToken(reserve.debtToken).burn(params.agent, repaid);
    ```

**`realizeInterest`**: Realize interest for an asset

```solidity
function realizeInterest(ILender.LenderStorage storage $, address _asset)
    external
    returns (uint256 realizedInterest)
```

**`realizeRestakerInterest`**: Realize interest for restaker debt of an agent for an asset

```solidity
function realizeRestakerInterest(ILender.LenderStorage storage $, address _agent, address _asset)
    public
    returns (uint256 realizedInterest)
```

Unrealized interest support: Can track interest that couldn't be realized due to liquidity constraintss

#### Admin Functions

**`setInterestReceiver`:** Set Interest Receiver for an asset

```solidity
function setInterestReceiver(address _asset, address _interestReceiver) external checkAccess(this.setInterestReceiver.selector)
```

**`setMinBorrow`:** Set minimum borrow amount for an asset

```solidity
function setMinBorrow(address _asset, uint256 _minBorrow) external checkAccess(this.setMinBorrow.selector)
```

#### View Functions

**`accruedRestakerInterest`**: Accrued Restaker Interest

```solidity
function accruedRestakerInterest(address _agent, address _asset) external view returns (uint256 accruedInterest)
```

**`unrealizedInterest`**: Unrealized Restaker Interest

```solidity
function unrealizedInterest(address _agent, address _asset) external view returns (uint256 _unrealizedInterest)
```

**`maxRealization`**: Calculates the maximum vault interest that can be realized for an asset

```solidity
function maxRealization(address _asset) external view returns (uint256 _maxRealization)
```

**`maxRestakerRealization`**: Calculates the maximum restaker interest that can be realized for an agent

```solidity
function maxRestakerRealization(address _agent, address _asset)
    external
    view
    returns (uint256 newRealizedInterest, uint256 newUnrealizedInterest)
```

* `newRealizedInterest`: Maximum realizable interest
* `newUnrealizedInterest`: Interest that will become unrealized

