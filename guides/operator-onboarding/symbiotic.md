# Symbiotic

Registering to Cap's Symbiotic Network as an operator is very straightforward: simply provide the operator's Ethereum address, referred to _agent_ in Cap's system, to&#x20;

1. Cap's onboarding team: registers the Symbiotic Vault to Cap's system with loan parameters such as Loan-to-Value, Liquidation Threshold and restaker fees.
2. Delegator you wish to be receiving stake from: the delegator will then create the agent-specific Symbiotic Vault

{% hint style="warning" %}
To isolate slashing risk, each operator can only receive effective stake from one Symbiotic Vault. Delegations from other Vaults will not count as collateral. The operator must create a new address for each new Vault. Deployed addresses cannot be changed.
{% endhint %}

Under the hood, the onboarding process happens as the following:

1. The delegator first creates a vault via the [CapSymbioticVaultFactory](https://github.com/cap-labs-dev/cap-contracts/blob/1064b6a969d55c822dcf0b2c4b733ceb4118737e/contracts/delegation/providers/symbiotic/CapSymbioticVaultFactory.sol#L57) including the agent address. The call will register the operator in Symbiotic's [Operator Registry](https://docs.symbiotic.fi/modules/registries/#2-operatorregistry) and complete the [Operator to Network Opt-in](https://docs.symbiotic.fi/modules/registries/#2-operator-to-network-opt-in) and [Operator to Vault Opt-in](https://github.com/cap-labs-dev/cap-contracts/blob/main/contracts/delegation/providers/symbiotic/SymbioticNetwork.sol#L77) process via the [SymbioticOperator](https://github.com/cap-labs-dev/cap-contracts/blob/1064b6a969d55c822dcf0b2c4b733ceb4118737e/contracts/delegation/providers/symbiotic/SymbioticOperator.sol#L4) contract.&#x20;
2. Once the Vault is deployed, Cap will register the agent via to Cap's system via the [addAgent](https://github.com/cap-labs-dev/cap-contracts/blob/1064b6a969d55c822dcf0b2c4b733ceb4118737e/contracts/interfaces/ISymbioticAgentManager.sol#L45) function. The function takes in the following struct:

```solidity
struct AgentConfig {
    address agent;
    address vault;
    address rewarder;
    uint256 ltv;
    uint256 liquidationThreshold;
    uint256 delegationRate;
}
```

This function will add the agent to the [Delegation](https://github.com/cap-labs-dev/cap-contracts/blob/1064b6a969d55c822dcf0b2c4b733ceb4118737e/contracts/delegation/Delegation.sol) contract, register the Vault and Agent to Cap's [Symbiotic Network Middleware](https://github.com/cap-labs-dev/cap-contracts/blob/1064b6a969d55c822dcf0b2c4b733ceb4118737e/contracts/delegation/providers/symbiotic/SymbioticNetworkMiddleware.sol) and set restaker rates

Delegations will take up to 6 days to be effective stake. Operators can start borrowing via the Asset page on Cap's app.&#x20;
