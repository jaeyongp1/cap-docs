# Oracles

The Oracle module in CAP is responsible for providing reliable, up-to-date price and rate data to the protocol. It acts as the backbone for all value calculations, including minting, burning, borrowing, and liquidation processes.&#x20;

## Oracle Data Sources

The protocol leverages multiple oracle sources for different types of data:

* [**RedStone Oracles**](https://www.redstone.finance/): Primary source for reserve asset pricing ([USDC, USDT, pyUSD & cUSD](https://app.redstone.finance/app/feeds/?page=1\&sortBy=popularity\&sortDesc=false\&perPage=32\&networks=1))
* [**Chainlink Oracles**](https://chain.link/): Used for delegation asset pricing (wstETH, wBTC) on shared security side
* [**Cap Token Adapter**](https://github.com/cap-labs-dev/cap-contracts/blob/1064b6a969d55c822dcf0b2c4b733ceb4118737e/contracts/oracle/libraries/CapTokenAdapter.sol): Calculates weighted average of underlying basket for cUSD pricing
* [**Staked Cap Adapter**](https://github.com/cap-labs-dev/cap-contracts/blob/1064b6a969d55c822dcf0b2c4b733ceb4118737e/contracts/oracle/libraries/StakedCapAdapter.sol): Accounts for accrued yield and cUSD price for stcUSD pricing
* [**Aave Adapter**](https://app.gitbook.com/u/YhHCfPpqr7SuXN3B26Zp7K85kgr1): Fetches current borrow rates from external markets
* [**Vault Adapter**](https://app.gitbook.com/u/YhHCfPpqr7SuXN3B26Zp7K85kgr1): Calculates utilization-based interest rates

## Architecture

The Cap oracle system uses a unified approach where both price and rate oracles are combined into a single [Oracle](https://github.com/cap-labs-dev/cap-contracts/blob/1064b6a969d55c822dcf0b2c4b733ceb4118737e/contracts/oracle/Oracle.sol) contract:

```solidity
contract Oracle is IOracle, UUPSUpgradeable, Access, PriceOracle, RateOracle
```

1. **Oracle Contract** - Main contract combining PriceOracle and RateOracle functionality
2. **PriceOracle Module** - Handles asset price data with dual oracle configuration
3. **RateOracle Module** - Manages interest rate data for lending operations
4. **Modular Adapters** - Pluggable adapters for different data sources
5. **Access Control** - Role-based access control for oracle configuration

### Price Oracle

The [price oracle](https://github.com/cap-labs-dev/cap-contracts/blob/1064b6a969d55c822dcf0b2c4b733ceb4118737e/contracts/oracle/PriceOracle.sol) provides asset price data with the following features:

* **Primary and Backup Sources**: Dual oracle configuration for reliability
* **Staleness Protection**: Configurable staleness periods for each asset
* **Modular Adapters**: Pluggable price source adapters
* **Error Handling**: Graceful fallback to backup sources

**`getPrice`**: retrieves price, falls back to backup oracle if prices are stale. If both timestamps revert, then relevant protocol functions are disabled until updated.

```solidity
function getPrice(address _asset) external view returns (uint256 price, uint256 lastUpdated)
```

* **Returns**: Current price and last updated timestamp

### Rate Oracle

The [rate oracle](https://github.com/cap-labs-dev/cap-contracts/blob/1064b6a969d55c822dcf0b2c4b733ceb4118737e/contracts/oracle/RateOracle.sol) provides interest rate data for lending operations:

* **Market Rates**: Returns the current borrow rate from external markets (e.g., Aave). Uses the [AaveAdapter](https://github.com/cap-labs-dev/cap-contracts/blob/1064b6a969d55c822dcf0b2c4b733ceb4118737e/contracts/oracle/libraries/AaveAdapter.sol) to fetch and relay the rate
* **Utilization Rates**: Returns the interest rate based on asset utilization
* **Benchmark Rates**: Returns the Cap’s minimum rate floors, set by admin
* **Restaker Rates**: Agent-specific fixed delegation rates

#### Core Functions

**`marketRate`**: Fetches current market borrow rate for an asset

```solidity
function marketRate(address _asset) external returns (uint256 rate)
```

**`utilizationRate`**: Fetches utilization-based interest rate for an asset

```solidity
function utilizationRate(address _asset) external returns (uint256 rate)
```

**`benchmarkRate`**: Gets the minimum interest rate floor for an asset

```solidity
function benchmarkRate(address _asset) external view returns (uint256 rate)
```

**`restakerRate`**: Gets the restaker rate for a specific agent

```solidity
function restakerRate(address _agent) external view returns (uint256 rate)
```

**Admin Functions**

* **`setPriceOracleData`**: Sets primary oracle data for an asset
* **`setPriceBackupOracleData`**: Sets backup oracle data for an asset
* **`setStaleness`**: Sets staleness period for asset prices in seconds
* **`setMarketOracleData`**: Sets market rate oracle data for an asset
* **`setUtilizationOracleData`**: Sets utilization rate oracle data for an asset
* **`setBenchmarkRate`**: Sets minimum interest rate floor for an asset
* **`setRestakerRate`**: Sets restaker rate for an agent

**View Functions**

* **`priceOracleData`**: Gets primary oracle data for an asset
* **`priceBackupOracleData`**: Gets backup oracle data for an asset
* **`staleness`**: Gets staleness period for an asset
* **`marketOracleData`**: Gets market rate oracle data for an asset
* **`utilizationOracleData`**: Gets utilization rate oracle data for an asset

### Data Structures

#### PriceOracleStorage

```solidity
struct PriceOracleStorage {
    mapping(address => IOracleTypes.OracleData) oracleData;        // Primary oracle data per asset
    mapping(address => IOracleTypes.OracleData) backupOracleData;  // Backup oracle data per asset
    mapping(address => uint256) staleness;                        // Staleness period per asset
}
```

#### RateOracleStorage

```solidity
struct RateOracleStorage {
    mapping(address => IOracleTypes.OracleData) marketOracleData;      // Market rate oracle data per asset
    mapping(address => IOracleTypes.OracleData) utilizationOracleData; // Utilization rate oracle data per asset
    mapping(address => uint256) benchmarkRate;                        // Benchmark rate per asset
    mapping(address => uint256) restakerRate;                         // Restaker rate per agent
}
```

#### OracleData

```solidity
struct OracleData {
    address adapter;  // Adapter contract address for calculation logic
    bytes payload;    // Encoded call to adapter with all required data
}
```
