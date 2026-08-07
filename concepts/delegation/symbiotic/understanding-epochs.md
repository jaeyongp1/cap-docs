# Understanding Epochs

### 1. Primer: dependency chain

Cap credit is backed by restaker delegation held in a Symbiotic vault dedicated to your agent. Recovery on a bad loan means slashing that delegation. Three facts follow from how Symbiotic implements slashing, and together they determine everything else in this document.

**Slashing is timestamp-addressed.** A slash calls `slashableStake(subnetwork, operator, captureTimestamp)` and executes against the stake that existed at that timestamp, not against the current balance.

**The capture timestamp is only valid for one Symbiotic vault epoch.** Outside that range there is nothing to slash and the recovery path fails.

**Therefore borrow capacity cannot be "what is delegated."** It has to be _what is slashable at a timestamp Cap can actually cite right now_. That quantity is what Cap calls **coverage**, and it is the input to your LTV and health factor.

The Cap epoch exists to supply that timestamp. It is a protocol-wide checkpoint cadence that gives every agent a capture timestamp which is deterministic, derivable on-chain by any liquidator without off-chain input, and guaranteed to sit inside Symbiotic's window.

### 2. Two epoch clocks

<table><thead><tr><th width="138.3046875"></th><th>Symbiotic vault epoch</th><th>Cap delegation epoch</th></tr></thead><tbody><tr><td><strong>Duration</strong></td><td>7 days</td><td>3 days</td></tr><tr><td><strong>Enforced by</strong></td><td>Vault config; Cap rejects vaults below 7 days at registration</td><td><code>Delegation.epochDuration()</code></td></tr><tr><td><strong>Determines</strong></td><td>Width of the valid capture-timestamp window</td><td>Which checkpoint <code>slashTimestamp</code> falls back to</td></tr><tr><td><strong>Phase</strong></td><td>Vault creation time</td><td>Absolute Unix time — boundaries at <code>timestamp % 259200 == 0</code></td></tr></tbody></table>

`slashTimestamp` is never more than **two Cap epochs (6 days)** old, against a **7-day** Symbiotic window. The one-day margin is the reason Cap enforces a 7-day minimum vault epoch and refuses to register anything shorter.

### 3. slashTimestamp

`slashTimestamp(agent) = max( (epoch() − 1) × epochDuration , lastBorrow − 1 )`

**The start of the previous Cap epoch is the floor.** The timestamp is always inside the Symbiotic window, and is what a dormant agent is measured against. Its age oscillates between 3 and 6 days as the current epoch advances.

**`lastBorrow − 1`** is the fast path. Cap writes `lastBorrow = block.timestamp` on every borrow. The instant before a borrow is by construction a moment when sufficient coverage existed, so it is both safe and far more recent. The `− 1` is required because Symbiotic rejects `captureTimestamp == now`; Cap also decrements defensively if the computed value lands on the current block.

### 4. Calculating coverage

`Delegation.coverage()` does not read a single value. It takes the minimum of four measurements below to state what is recoverable.

<table><thead><tr><th width="45.7890625">#</th><th>Measurement</th><th>Timestamp</th><th>Denies credit for</th></tr></thead><tbody><tr><td>1</td><td><code>slashableStake</code></td><td>Start of <strong>current</strong> Cap epoch</td><td>Delegation added mid-epoch, not yet checkpointed</td></tr><tr><td>2</td><td><code>slashableStake</code></td><td><strong><code>slashTimestamp</code></strong></td><td>Coverage that could not actually be slashed today</td></tr><tr><td>3</td><td><code>stakeAt</code> (active)</td><td><strong>Now</strong></td><td>Stake already queued for withdrawal</td></tr><tr><td>4</td><td><code>slashableStake</code></td><td><strong><code>now − 1</code></strong></td><td>As #3, one block back</td></tr></tbody></table>

**Quantities are snapshotted; prices are not.** Each measurement snapshots the collateral _amount_ at its timestamp, then values it at the **current** oracle price. Price movement therefore transmits to coverage with zero lag. The epoch mechanism lags quantity only.

**Active and slashable stake diverge.** A withdrawal request removes stake from _active_ immediately while leaving it _slashable_ until the withdrawal clears. Measurements 1, 2 and 4 read slashable stake; measurement 3 reads active stake. The `min()` means a withdrawal request cuts your capacity on request, not on settlement.

### 5. Refreshing slashTimestamp by borrowing

Every borrow writes `lastBorrow`. It is the only borrower-controlled input to the epoch machinery that can update the coverage amount of the restaker.

New delegation counts only once **both** lagging snapshots postdate it. They clear by different means:

* **The slash-timestamp snapshot you can move yourself.** Any borrow at or above the reserve's `minBorrow` sets `lastBorrow = block.timestamp`, pulling `slashTimestamp` to one second ago. Effective in the same block, regardless of draw size.
* **The epoch snapshot you cannot.** It is pinned to the current epoch boundary and advances only when the clock crosses the next one.

Sequence after a restaker increases your delegation:

<table><thead><tr><th width="78.21484375">Step</th><th>Action</th><th>Result</th></tr></thead><tbody><tr><td>1</td><td>Wait for the next epoch boundary</td><td>Epoch snapshot now postdates the increase</td></tr><tr><td>2</td><td>Borrow ≥ <code>minBorrow</code> immediately after</td><td><code>slashTimestamp</code> now postdates the increase</td></tr><tr><td>3</td><td>—</td><td>Full coverage live</td></tr></tbody></table>

Borrowing before the boundary accomplishes nothing — the epoch snapshot still predates the increase, and `min()` still selects it. Crossing the boundary without borrowing does work, but leaves `slashTimestamp` sitting on the previous-boundary floor, which costs another 3 days.

**Constraints on the refresh**

* **Borrows validate against pre-borrow coverage.** `validateBorrow` sizes against `maxBorrowable` as it currently stands, so the refresh draw cannot itself consume the capacity it unlocks. Refresh with a minimal draw, then size up.
* **Repayment does not write `lastBorrow`.** Only `borrow` does.
* **Over-refreshing is free.** A `lastBorrow` older than the previous epoch boundary is simply ignored — the floor takes over — so there is no penalty for refreshing more often than strictly necessary.

### 6. Reference

| Query                                  | Call                                                               |
| -------------------------------------- | ------------------------------------------------------------------ |
| Current epoch index                    | `Delegation.epoch()`                                               |
| Capture timestamp in force             | `Delegation.slashTimestamp(agent)`                                 |
| Coverage driving your capacity         | `Delegation.coverage(agent)`                                       |
| Slashable collateral at that timestamp | `Delegation.slashableCollateral(agent)`                            |
| LTV / liquidation threshold (ray)      | `Delegation.ltv(agent)` · `Delegation.liquidationThreshold(agent)` |
