# Operator Onboarding

The following outlines the process for onboarding as an operator to borrow from Cap.

### 1. Setup

Ensure you have:

* **Delegator Profile**: the list of active delegators can be found [here](https://cap.app/delegators).&#x20;
* **Restaker Rate**: Operators and restakers should agree on a fixed restaker rate to pay the Delegator
* **Wallet**: an Ethereum Externally Owned Account (EOA) or multisig wallet to serve as your operator identity

{% hint style="info" %}
Operators must generate a new wallet for each new delegator to receive stake from. Addresses cannot be changed after deployment. For best practices, we recommend using a multisig for the address.
{% endhint %}

### 2. Register to a SSN of choice&#x20;

Currently, Cap supports the following Shared Security Networks.

1. [Symbiotic](symbiotic.md)
2. EigenLayer

### 3. Complete Legal Agreements (Optional)

Operators and delegators may enter into legal agreements outlining terms of delegation, responsibilities, and compliance.

### 4. Participate in Loan Activity&#x20;

At this step, operators may engage in borrowing activities. They can verify delegations and borrowing power from Cap's Operator Page.

Review the risk parameters prior to loan:

* **LTV (Loan-to-Value)**: Maximum borrowing capacity (e.g., 50% = 0.5e27)
* **Liquidation Threshold**: Health factor at which liquidation can occur (e.g., 80% = 0.7e27)
* **LTV Buffer**: Minimum gap between LTV and liquidation threshold (10% = 0.05e27)

### 5. Updating Parameters

Should delegators and operators wish to change loan parameters, please contact the Cap team to do so.

Namely, delegation admin can update the restaker rate via the [setRestakerRate](https://github.com/cap-labs-dev/cap-contracts/blob/1064b6a969d55c822dcf0b2c4b733ceb4118737e/contracts/delegation/Delegation.sol) function, and the LTV and LT via the [modifyAgent](https://github.com/cap-labs-dev/cap-contracts/blob/1064b6a969d55c822dcf0b2c4b733ceb4118737e/contracts/delegation/Delegation.sol#L97) function.
