# FAQs

## Operator Control and Restrictions

**Q: Do operators have full control over collateral once disbursed?**

No. The operator set will be comprised of regulated financial institutions that enter into legal agreements with restakers to restrict permissible strategies.&#x20;

**Q. How does the protocol ensure there are no risky strategies?**

Restakers bear direct risk for their delegation choices. By underwriting specific operators, restakers incentivize due diligence on strategy viability.

**Q: Are there caps on collateral lent to operators?**

Yes. Delegation limits and nominal exposure thresholds prevent concentration risk.

## Loan Security

**Q: Does the protocol impose minimum overcollateralization requirements on restakers?**&#x20;

Yes, the protocol checks the collateralization ratio before the operator can borrow. The overcollateralization ratio is different for every collateral asset and is set conservatively to prevent any unexpected liquidations.

**Q: Which assets are accepted as restaker collateral?**

Only blue-chip cryptocurrencies: ETH, wBTC, Liquid Staking Tokens (LSTs), and stablecoins.

**Q: What happens if restakers withdraw their delegation mid-loan?**&#x20;

Restakers and operators should have a prior agreement when restakers intend to remove delegation for position unwinding. Unilateral withdrawals will trigger slashing. Note that there is a built-in buffer period in Shared Security Networks to mitigate an accidental withdrawal.

## Shared Security Network

**Q. How is Slashing and Redistribution used in Cap?**

Malicious or undercollateralized operators are autonomously slashed, where restaker delegations are redistributed back to the stablecoin holders.

**Q. How does Cap differ from other restaking protocols?**

Cap replaces passive Proof-of-Stake rewards with productivity-based incentives:

* Restakers earn premiums tied to operator performance.
* Operators retain surplus yield, fostering competitive strategy innovation.

Cap takes an innovative approach of rewarding restakers on an operator basis, where counterparty risk is established one-to-one with the operator they are underwriting. This architecture allows operators to take more agency and be rewarded for their productivity.&#x20;

For details, please check out our [Productivity Based Incentives](https://mirror.xyz/0x83c21bb4Bf0EC116f5a1945AaeF847Fe3b321B32/Xg7gCAqgmcxgXhuwaWXVJDSCEIMn3_dewNWAie2Tn_k) article.
