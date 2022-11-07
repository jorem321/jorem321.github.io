Lido.sol is the core contract governing the logic of the liquid staking pool. It takes care of several tasks, such as:

- Handling Ether deposits, minting stETH, and paying stETH rewards.
- Applying fees and handling the treasury
- Withdrawals (not yet implemented)
- Delegating funds to node operators
- Accepting updates about the beacon chain’s state from the oracle contract.

*Note: Lido.sol inherits the abstract contract stETH.sol. Since the latter is not deployable, we will be treating the functions in stETH.sol as part of Lido.sol.*

## Depositing Ether and rebasing rewards: shares vs stETH

A prospective liquid staker can deposit ETH and mint stETH in two ways:
``
1. Depositing directly to the Lido contract. This will call the fallback function `function() external payable`
2. By calling the function `submit(address _referral)`. Here, the staker can indicate a referral address, who will be paid 0.25% of the ETH staked in LDO tokens as reward.

Either way, the internal function `function _submit(address _referral) internal returns (uint256)` gets called, which will take the following steps:

- A number of “shares” will be internally assigned to the account sending the ETH. These shares work as a bookkeeping mechanism to make sure all users are paid the same corresponding APR (we will show this below).
    
    Let shares for an account $i$ be denoted $s_i$. Let $\xi$ represent the amount of Ether deposited by the user, and $\Xi(t)$ denote the total amount of Ether in possession by the protocol on day $t$ (both on the Lido contract and staked on the beacon chain). Then shares for the new user $n+1$ are assigned via the [formula](https://github.com/lidofinance/lido-dao/blob/df95e563445821988baf9869fde64d86c36be55f/contracts/0.4.24/StETH.sol#L315)
    
    $$
    \begin{equation} s_{n+1} = \frac{\xi}{\Xi(t)} \left( \sum_{i=1}^n s_i \right) \end{equation}
    $$
    
    and stored in the data structure `mapping (address => uint256) private shares` . 
    
    - We can think of this intuitively as a user buying shares of a company with his ETH, at a unit price $\frac{\text{total pooled ETH}}{\text{total shares}} = \frac{\Xi}{\sum s_i}$. The numerator (i.e. share price) increases as the protocol earns staking rewards.
- Once a user $i$ has a number of shares in the Lido contract, they are shown an stETH balance corresponding to
    
    $$
    \begin{equation}\text{stETH}_i(t) = 
    s_i \times \text{share price} = s_i \times\frac{\Xi(t)}{\sum_{j} s_j} \end{equation}
    $$
    
    By combining equations (1) and (2) it is easy to see that $\text{stETH}_n(t) = \xi$ at the time of minting.
    
- The contract holds the deposited ETH under custody until it can be used to create a new 32-ETH validator, which is delegated to a node operator. This is done by calling `function depositBufferedEther(uint256 _maxDeposits) external`.

How does the stETH balance of a user change as a function of time? Consider equations (1) and (2):

- The share-price fraction, $\frac{\Xi(t)}{\sum_{j} s_j}$, does not change as a function of new stETH deposits. We can see this by rearranging equation (1) as

$$
\frac{s_{n+1}}{\xi}
 = \frac{\sum_{i=1}^n s_i}{\Xi} = \frac{\sum_{i=1}^{n+1} s_i}{\Xi + \xi}.
$$

- As such, the share-price fraction only grows from staking rewards in the numerator $\Xi(t)$, which are in proportion to the APR. We conclude that stETH grows with the same APR as $\Xi(t)$…but is this correct?
- **Remark**: We know that stETH’s APR is actually 90% of the full APR amount. The other 10% is split equal parts between treasury and node operators. What is missing in this analysis to account for the other 10%? See next section.

### Lido protocol fees

The 10% cut taken for treasury and node operators is handled by generating new shares. Every day, `function distributeFee(uint256 _totalRewards) internal` is called, which generates the [exact amount of new shares](https://github.com/lidofinance/lido-dao/blob/master/contracts/0.4.24/Lido.sol#L815) required to subtract 10% value from the newly generated staking rewards. Half of these shares go to the treasury, the other half are distributed amongst node operators according to their number of validators.

### Transferring stETH

stETH transfers are not really such. Instead, when a user transfers stETH via the function `transfer`, they are transferring Lido shares, with the corresponding share-price conversion handled by the Lido contract. 

### Getting your ETH back

Withdrawals are not currently available on the beacon chain, and will not be until post-merge. Therefore, Lido has not implemented a mechanism that can redeem stETH into ETH. From conversations with the team, this redemption will be done in a 1:1 fashion, and there will be a waiting time corresponding to the beacon-chain exit queue.

However, users can still go to secondary markets and sell their stETH for ETH, at a market price very close to the 1:1 ratio.

(Note that Lido contracts are upgradable via the proxy pattern. For example, the proxy for the contract we are studying can be found [here](https://etherscan.io/address/0xb8FFC3Cd6e7Cf5a098A1c92F48009765B24088Dc#code).)

### Delegating funds to node operators

We know that Ether gets stored in the Lido contract as a buffer. We wait for 32 Ether or more to be in the contract’s wallet, so that new validators can be created. How is this done?

A Lido member—owning an account with `DEPOSIT_ROLE` credentials—calls `function depositBufferedEther() external` or `function depositBufferedEther(uint256 _maxDeposits) external` (the difference being whether we limit the number of validators created with the `_maxDeposits` variable). Then, the following function calls happen.

- `_ETH2Deposit(numDeposits)`: Fetches enough operators to cover numDeposits. For each of the operators, we also fetch [the required credentials for initializing a validator](https://github.com/ethereum/annotated-spec/blob/master/phase0/beacon-chain.md#aside-note-on-the-deposit-process), namely:
    - `pubkey`: a viable new public signing key for the new validator.
    - `signature`: concatenate the signing key and the withdrawal key (along with the amount of Ether to be deposited, which should be 32), and sign them with the signing key. This signature is required to start new validators as per the beacon chain spec.
- For each of the pairs `(pubkey, signature)`, the function `_stake(pubkey, signature)` is called, initializing a new validator with the corresponding signing key.
    - Note: the withdrawal key stays in Lido’s power.

### Receiving updates from the beacon chain

Every day, the LidoOracle contract will call `function handleOracleReport(uint256 _beaconValidators, uint256 _beaconBalance) external whenNotStopped`. This function will update $\Xi(t)$, the total Ether’s balance controlled by the protocol. We have 2 cases here:

- $\Xi(t+1)>\Xi(t)$: then the difference $\Xi(t+1)-\Xi(t)$ is distributed as rewards in the form of stETH tokens. 90% of the difference is reflected as rewards for all stETH holders, 5% is minted as new shares for the node operators, and 5% goes to the treasury.
- $\Xi(t+1)\leq\Xi(t)$: this can be the case due to slashing events. In this scenario, Lido can opt to burn shares from the treasury by calling `function burnShares(address _account, uint256 _sharesAmount)`, so that shares do not fall in value. This function has to be called manually—it is not scripted to run on a negative oracle report.

---

# Inputs

| name | type | description |
| --- | --- | --- |
| _numDeposits | uint256 | Number of deposits to perform |
| _sharesToDistribute | uint256 | amount of shares to distribute |

# Internal constants

| name | type | Value |
| --- | --- | --- |
| NODE_OPERATORS_REGISTRY_POSITION | bytes32 | keccak256("lido.Lido.nodeOperatorsRegistry") |

# Functions

### `getOperators()`

```solidity
function getOperators() public view returns (INodeOperatorsRegistry) {
        return INodeOperatorsRegistry(NODE_OPERATORS_REGISTRY_POSITION.getStorageAddress());
}
```

Returns `INodeOperatorsRegistry` which is an interface for the `NodeOperatorRegistry` contract

This interface is used to get the functions

- `trimUnusedKeys()`
- `assignNextSigningKeys(_numDeposists)`
- `getRewardsDistribution(_sharesToDistribute)`
