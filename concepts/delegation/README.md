# Delegation

The Delegation module serves as a middleware between Shared Security Networks and the lending protocol, enabling:

* **Operator Management:** Register SSN Operators to Cap, setting their LTV/liquidation ratio and restaker fees
* **Collateral Provision**: Delegators can collateralize Operators for borrowing capacity
* **Slashing:** Execute slashing of delegated collateral during liquidations
* **Reward Distribution**: Distribute rewards to delegators proportionally to their coverage

Cap supports multiple SSNs. Currently integrated SSNs include:

* [Symbiotic](symbiotic.md)
* EigenLayer

## Overview of Cap SSN Requirements

### Isolated Coverage

Delegation coverage is isolated both on a Network level and an Operator level. Each Operator is uniquely mapped to a SSN and a Delegator. Each Operator receives unique, isolated stake from a single Delegator such that slashing risk is not shared.

Network level isolation means that delegations cannot be "restaked" across multiple slashable restaking protocols - because delegations are used as collateral in Cap, a slashing event in a different Network other than Cap would result in bad debt for Cap.&#x20;

Similarly, restaking across multiple Operators within Cap is not allowed. When multiple Delegators are delegating to the same operator, a withdrawal from one of the Delegators can result in a slashing event for other Delegators.&#x20;

In other words, a Delegator's stake can only be used for a single Operator in Cap. Alternatively, an Operator cannot receive stake from multiple Delegators. If the Operator were to receive delegations from a new Delegator, then the Operator can simply create a new Ethereum address to do so.

### Restaker Fees

Each Operator-Delegator agree upon a predetermined fee for restaking. As such, each Operator-Delegator have an unique fixed yearly rate that can be updated via Admin controls. The rates are set in the respective SSN upon deployment.&#x20;

Restaker fees are distributed per each SSN's reward handler.&#x20;

### Slashing and Redistribution

Slashing conditions in Cap are objective: they are akin to liquidations and thus triggered when the health factor of the Operator drops below the liquidation threshold. Liquidations can be called permissionlessly, and verified onchain. Slashed funds are redistributed back to the liquidators, ensuring the cUSD is always backed 1:1.

To incentivize liquidations, slashing and redistribution happen instantly. There are no veto committees nor delay periods for slashing to occur, nor for the slashed funds to be redistributed.&#x20;

### Whitelisted Participants

While the protocol can function fully autonomously, to prevent malicious actors from attacking the protocol, Cap will initially whitelist Delegators and Operators. Delegators and Operators that seek to join the protocol should refer to the Onboarding Guides to do so.

## Architecture

1. **Delegation Contract** - Main contract managing agents, networks, and delegation operations
2. **SymbioticNetworkMiddleware** - Adapter for Symbiotic network integration
3. **SymbioticNetwork** - Network management and operator deployment

### Delegation Contract

#### Core Functions

**`slash`**: Executes slashing of delegated collateral during liquidations

```solidity
function slash(address _agent, address _liquidator, uint256 _amount) external checkAccess(this.slash.selector)
```

* `_agent`: Agent address to slash
* `_liquidator`: Address to receive slashed collateral
* `_amount`: USD value of delegation to slash

**`distributeRewards`**: Distributes rewards to networks covering an agent

```solidity
function distributeRewards(address _agent, address _asset) external
```

* `_agent`: Agent address for reward distribution
* `_asset`: Reward token address

#### Admin Functions

**`addAgent`**: Adds a new agent (operator) to the delegation system

```solidity
function addAgent(address _agent, address _network, uint256 _ltv, uint256 _liquidationThreshold) external checkAccess(this.addAgent.selector)
```

* `_agent`: Agent address to add
* `_network`: Network address for the agent
* `_ltv`: Loan-to-value ratio for the agent
* `_liquidationThreshold`: Liquidation threshold for the agent

**`modifyAgent`**: Modifies an existing agent's configuration

```solidity
function modifyAgent(address _agent, uint256 _ltv, uint256 _liquidationThreshold) external checkAccess(this.modifyAgent.selector)
```

* `_agent`: Agent address to modify
* `_ltv`: New loan-to-value ratio
* `_liquidationThreshold`: New liquidation threshold

**`registerNetwork`**: Registers a new shared security network

```solidity
function registerNetwork(address _network) external checkAccess(this.registerNetwork.selector)
```

* `_network`: Network address to register

**`setLtvBuffer`**: Sets the LTV buffer between LTV and liquidation threshold

```solidity
function setLtvBuffer(uint256 _ltvBuffer) external checkAccess(this.setLtvBuffer.selector)
```

**`setFeeRecipient`**: Sets the fee recipient for cases with no coverage

```solidity
function setFeeRecipient(address _feeRecipient) external checkAccess(this.setFeeRecipient.selector)
```

#### View Functions

* **`coverage`**: Gets the amount of delegation available to back an agent's borrows
* **`slashableCollateral`**: Gets the amount of slashable collateral for an agent
* **`slashTimestamp`**: Gets the appropriate slash timestamp for an agent
* **`ltv`**: Gets the loan-to-value ratio for an agent
* **`liquidationThreshold`**: Gets the liquidation threshold for an agent
* **`agents`**: Gets all registered agent addresses
* **`networkExists`**: Checks if a network is registered
* **`epochDuration`**: Gets the epoch duration in seconds
* **`epoch`**: Gets the current epoch
* **`ltvBuffer`**: Gets the LTV buffer
* **`feeRecipient`**: Gets the fee recipient address

### Data Structures

Delegation Storage

```solidity
struct DelegationStorage {
    EnumerableSet.AddressSet agents;                    // Registered agent addresses
    mapping(address => AgentData) agentData;            // Per-agent configuration
    EnumerableSet.AddressSet networks;                  // Registered network addresses
    address oracle;                                     // Oracle for price feeds
    uint256 epochDuration;                             // Epoch duration in seconds
    uint256 ltvBuffer;                                 // LTV buffer from liquidation threshold
    address feeRecipient;                              // Fee recipient address
}
```

Agent Data

```solidity
struct AgentData {
    address network;              // Associated network address
    uint256 ltv;                 // Loan-to-value ratio (ray)
    uint256 liquidationThreshold; // Liquidation threshold (ray)
    uint256 lastBorrow;          // Last borrow timestamp
}

```
