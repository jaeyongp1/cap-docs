# Delegator Onboarding

The following outlines the process for onboarding delegators to Shared Security Networks (SSNs) in Cap's protocol. Onboarded delegators may delegate to operators approved in the Cap system.

### 1. Setup

Ensure you have:

* **Operator Address:** Operator's Ethereum address that will receive delegations. The operator must be already registered in Cap's system.
* **Restaker Rate**: Agree on a fixed rate to receive from the Operator
* **Collateral Asset:** The asset to be used as vault collateral (ETH/BTC-denominated ERC20s)

### 2. Choose SSN

Select SSN of choice for delegations, and follow the corresponding onboarding guide to complete set up and start delegating.

1. [Symbiotic](symbiotic.md)
2. EigenLayer

Collateral management, i.e. delegations and withdrawals, are handled within each SSN. Delegators are advised to monitor delay periods specific to the SSN.

{% hint style="danger" %}
Withdrawing delegations immediately lowers coverage. Delegated assets in the withdrawal queue are slashable until the end of the withdrawal delay. Hence, a withdrawal below the liquidation threshold makes the delegation asset slashable.&#x20;
{% endhint %}

While a time buffer is in place to mitigate risk, it is recommended that Delegators whitelist depositors  prevent malicious/accidental withdrawals that may trigger unintended slashing.

### 3. Complete Legal Agreements (Optional)

Operators and Delegators may enter into legal agreements outlining terms of delegation, responsibilities, and compliance.
