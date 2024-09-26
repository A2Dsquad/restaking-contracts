# Sentra Layer smart contracts

## Development progress

### Core restaking

| **Core restaking features**  | **TO DO** | **In Progress** | PoC | **Refined** | **Audited** |
|------------------------------|-----------|-----------------|-----|-------------|-------------|
| Restake LSTs                 |           |                 | ✅   |             |             |
| Delegate to operators        |           |                 | ✅   |             |             |
| Integrate Fungible Assets    |           |                 | ✅   |             |             |
| Wrapper for Coins            |           |                 | ✅   |             |             |
| Queue withdrawals            |           |                 | ✅   |             |             |
| Complete queued withdrawals  |           |                 | ✅   |             |             |
| Slasher                      |           |                 | ✅   |             |             |
| Veto slashing                |           | 🚧               |     |             |             |
| Rewards Submission           |           |                 | ✅   |             |             |
| Rewards Distribution & Claim |           |                 | ✅   |             |             |
| Restake APT                  | 📝         |                 |     |             |             |
| Customizable Slasher         |           | 🚧               |     |             |             |

### Middleware for AVS ZKbridge service

| **Core AVS features**             | **TO DO** | **In Progress** | PoC | **Refined** | **Audited** |
|-----------------------------------|-----------|-----------------|-----|-------------|-------------|
| BLS registry                      |           |                 | ✅   |             |             |
| Tasks submission                  |           |                 | ✅   |             |             |
| Tasks verification                |           |                 | ✅   |             |             |
| Mint bridged tokens               |           |                 | ✅   |             |             |
| AVS registry                      |           |                 | ✅   |             |             |
| Flexible Task verification logics |           | 🚧               |     |             |             |

## Contracts

### Mainnet

// To be updated

### Devnet

- restaking: 
- avs middleware: 

## Structure

### Main components

### Objects / Structs

1. `restaking::staker_manager::StakerStore`:
- Created for each staker.
- Stores user shares for each staked asset.
- Contains queued withdrawal data.
- Stores the operator to whom the staker delegates.

```move
  struct StakerStore has key {
    delegated_to: address,
    cummulative_withdrawals_queued: u256,
    pool_list: SmartVector<Object<Metadata>>,
    nonnormalized_shares: SmartTable<Object<Metadata>, u128>
  }
```

2. `restaking::operator_manager::OperatorStore`:
- Created for each operator.
- Stores operator shares for each staked asset.
- Contains the AVSs secured by the operator.

```
  struct OperatorStore has key {
    nonnormalized_shares: SmartTable<Object<Metadata>, u128>,
    salt_spent: SmartTable<u256, bool>,
  }
```

3. `restaking::slasher::OperatorSlasherStore`:
- Created for each operator-asset pair.
- Manages operator slashing state and history.

```move
  struct SlashingRequestIds has drop, store{
    last_created: u32,
    last_executed: u32,
  }
  struct SlashingRequest has copy, drop, store {
    id: u32,
    slashing_rate: u64,
    scaling_factor: u64
  }
  struct OperatorSlashingStore has key {
    slashing_request_ids: SlashingRequestIds,
    slashing_requests: SmartTable<u64, SlashingRequest>,
    share_scaling_factor: u64,
    slashed_epoch_history: SmartVector<u64>,
  }
```

4. `restaking::earner_manager::EarnerStore`:
- Created for each earner (typically the staker).
- Stores the delegated claimer.
- Tracks cumulative claims over time.

```move
  struct EarnerStore has key {
    claimer: address,
    cummulative_claimed: SmartTable<Object<Metadata>, u64>,
  }
```

5. `restaking::withdrawal::WithdrawalStore`:
- A universal object that manages pending withdrawals.
- Tracks the minimum withdrawal delay for each staked asset.
- restaking::avs_manager::AVSStore:
- Created for each AVS.
- Stores the AVS’s submitted rewards.

```move
  struct Withdrawal has copy, drop, store {
    staker: address,
    delegated_to: address,
    withdrawer: address,
    nonce: u256,
    start_time: u64,
    tokens: vector<Object<Metadata>>,
    nonnormalized_shares: vector<u128>,
  }

  struct PendingWithdrawalData has drop, store {
    is_pending: bool,
    creation_epoch: u64,
  }

  struct WithdrawalConfigs has key {
    signer_cap: SignerCapability,
    min_withdrawal_delay: u64,
    pending_withdrawals: SmartTable<u256, PendingWithdrawalData>,
    token_withdrawal_delay: SmartTable<Object<Metadata>, u64>,
  }
```

6. `restaking::rewards_coordinator::RewardsStore`: A universal store for managing the distribution of rewards generated by AVSs.

```move
  struct EarnerMerkleTreeLeaf has copy, drop, store {
    earner: address,
    earner_token_root: vector<u8>
  }

  struct TokenTreeMerkleLeaf has copy, drop, store {
    token: Object<Metadata>,
    cummulative_earnings: u64
  }

  struct RewardsMerkleClaim has copy, drop, store {
    root_index: u64,
    earner_index: u32,
    earner_tree_proof: vector<u8>,
    earner_leaf: EarnerMerkleTreeLeaf,
    token_indices: vector<u32>,
    token_tree_proofs: vector<vector<u8>>,
    token_leaves: vector<TokenTreeMerkleLeaf>,
  }

  struct DistributionRoot has copy, drop, store {
    root: u256,
    rewards_calculation_end_time: u64,
    activated_at: u64,
    disabled: bool,
  }

  struct RewardsConfigs has key {
    signer_cap: SignerCapability,
    rewards_updater: address,
    activation_delay: u64,
    current_rewards_calculation_end_time: u64,
    global_operator_commission_bips: u16,
    distribution_roots: SmartVector<DistributionRoot>,
    rewards_for_all_submitter: SmartVector<address>,
  }
```

7. `middleware::service_manager::TaskStore`:
- Created for each cluster of AVS services.
- Stores necessary data for AVS task verification.
- Can be customized for each type of AVS service (e.g., verifying ETH -> Aptos bridging results).


### Modules

1. `restaking::staker_manager`:
- Manages all StakerStore objects.
- Handles stakers' deposits, withdrawals, and delegation.
- Ensures that stakers can interact seamlessly with operators.

2. `restaking::operator_manager`:
- Manages all OperatorStore objects.
- Calculates operator shares for each staked asset.
- Manages the registration and operations of operators.

3. `restaking::earner_manager`:
- Manages all EarnerStore objects.
- Defines who is the claimer of each staker’s earnings.
- Tracks cumulative claims over time.

4. `restaking::withdrawal`:
- Manages withdrawal operations, including queued withdrawals and undelegation of assets.
- Ensures compliance with minimum withdrawal delays.

5. `restaking::slasher`:
- Allows slashing committees to send slashing requests.
- Executes slashing actions on specific operators based on governance decisions.

6. `restaking::rewards_coordinator`:
- Facilitates rewards distribution and claim verification.
- Helps rewards managers submit distribution data.
- Enables claimers to claim their distributed rewards.

7. `restaking::registry_coordinator`:
- Handles AVS operator registrations.
- Ensures that only verified operators are allowed to secure AVSs.

8. `middleware::service_manager`:

- Tailored to each AVS service.
- Handles AVS task challenges.
- Verifies the submission and completion of AVS tasks (e.g., bridging transaction verifications from other blockchains).

9. `restaking::coin_wrapper`: As every asset in the smart contract system is treated as fungible_asset, all coins integrating with the system will be first wrapped into fungible_asset

10. Helper modules

- `restaking::epoch`: calculate epoch from timestamp (neccessary for slashing)

- `restaking::math`: math functions like bytes32-u256 converters

- `restaking::merkle_tree`: verify merkle proof (neccessary for rewards claim verification)

- `restaking::slashing_accounting`: calculate the shares of operators before / after slashed

### Public interface (entry functions)

#### Stake coins/assets

##### Module: `restaking::staker_manager`

1. Stake fungible asset
```move
  public entry fun stake_asset_entry(
    staker: &signer,
    token: Object<Metadata>,
    amount: u64
  ) acquires StakerStore, StakerManagerConfigs
```

Called by the staker to deposit a fungible asset into the protocol

`token`: the fungible asset metadata

`amount`: the deposit amount

2. Stake coins

```move
  public entry fun stake_coin_entry<CoinType>(
    staker: &signer,
    amount: u64
  )
```

Called by the staker to deposit a coin into the protocol

`amount`: the deposit amount of the coin

3. Delegate

``` move
public entry fun delegate(staker: &signer, operator: address) acquires StakerStore, StakerManagerConfigs
```

Called by the staker to delegate his stake to an operator

##### Module: `restaking::withdrawal`

1. Undelegate

```move
public entry fun undelegate(sender: &signer, staker: address) acquires WithdrawalConfigs
```
Called by the staker to undelegate his selected operator
The function can also be called by the operator to forge undelegation

2. Queue withdrawal

```move
  public entry fun queue_withdrawal(
    sender: &signer,
    tokens: vector<Object<Metadata>>,
    nonnormalized_shares: vector<u128>
  ) acquires WithdrawalConfigs
```

Called by the staker to queue a withdrawal of his staked assets

`tokens`: the list of withdrawn assets

`nonnomalized_shares`: the amount of withdrawn shares, corresponding to each asset in the `tokens`

3. Complete queued withdrawal

```move
  public entry fun complete_queued_withdrawal(
    sender: &signer,
    staker: address,
    delegated_to: address,
    withdrawer: address,
    nonce: u256,
    start_time: u64,
    tokens: vector<Object<Metadata>>,
    nonnormalized_shares: vector<u128>,
    receive_as_tokens: bool
  ) acquires WithdrawalConfigs
```

Called by the staker to complete his previous queued withdrawal

`staker`: the staker address

`delegated_to`: the delegated operator of the staker when at the queueing time

`withdrawer`: the address which is eligible to receive the withdrawn assets/shares

`nonce`: a number corresponding to a staker that will increment whenever that staker queues a withdrawal

`start_time`: the queueing time

`tokens`: the assets in the queued withdrawal

`nonnomarlized_shares`: the withdrawn shares of assets in the queued withdrawal

`receive_as_tokens`: if set `true`, the protocol will withdraw the specified tokens and transfer to the withdrawer, otherwise, the shares will be added back to the withdrawer.

##### Module: `restaking::slasher`

1. Request slashing

```move
  public entry fun request_slashing(
    sender: &signer,
    operator: address,
    tokens: vector<Object<Metadata>>,
    epoch: u64,
    scaling_factor: u64
  ) acquires OperatorSlashingStore, SlasherConfigs
```

Queue a slashing request to the specified operator

`operator`: the slashed operator

`tokens`: the slashed staked assets

`epoch`: indicates when the slashing will apply

`scaling_factor`: indicates how much operator's shares will be slashed

2. Execute slashing

```move
  public entry fun execute_slashing(
    operator: address,
    tokens: vector<Object<Metadata>>,
    epoch: u64
  ) acquires OperatorSlashingStore
```

Called by the slashing committe to apply the requested slashing when the time constraint for the requested slashing meets

`operator`: the slashed operator

`tokens`: the slashed assets

`epoch`: the epoch of the slashing request

##### Module: `restaking::eanrer_manager`

1. Set claimer for

```move
public entry fun set_claimer_for(sender: &signer, new_claimer: address) acquires EarnerStore, EarnerManagerConfigs
```

Called by the staker to designate who can claim rewards on his behalf

`new_claimer`: the claimer address

### Module: `restaking::avs_manager`

1. Submit rewards

```move
  public entry fun create_avs_rewards_submission(
    sender: &signer,
    tokens: vector<Object<Metadata>>,
    multipliers: vector<u64>,
    rewarded_token: Object<Metadata>,
    rewarded_amount: u64,
    start_time: u64,
    duration: u64,
  ) acquires AVSStore, AVSManagerConfigs
```

Called by the AVS to submit rewards, the payout asset will be transferred to the protocol treasury for later claiming, and the rewards information will be stored on-chain, queried by the rewards updater for off-chain distribution calculation.

`tokens`: staked assets eligible for claiming rewards

`multipliers`: indicates how much rewards will be distributed for holders of each staked asset.

`rewarded_token`: the payout asset

`rewarded_amount`: the total payout amount

`start_time`: when the rewards will be released


### Module: `restaking::rewards_coordinator`

1. Submit root

```move
  public entry fun submit_root(
    sender: &signer,
    root: u256,
    rewards_calculation_end_time: u64
  ) acquires RewardsConfigs
```

The distribution data is a Merkle Tree with leaves containing earners and distributed tokens, for efficient on-chain storing, only the tree root will be submitted and stored. This function is called by a trusted reward updater.

`root`: The distribution tree root

`rewards_calculation_end_time`: indicates when earners can claim the distribution, neccessary for recalculating in cases of mistakes.

2. Disable root

```move
  public entry fun disable_root(
    sender: &signer,
    root_index: u64
  ) acquires RewardsConfigs
```

Called by the rewards updater to invalidate a false submitted distribution root, this must be called before the `rewards_calculation_end_time` of the root passed.

`root_index`: the index of the disabled root

3. Process Claim

```move
  public entry fun process_claim(
    sender: &signer,
    recipient: address,
    root_index: u64,
    earner_index: u32,
    earner_tree_proof: vector<u8>,
    earner: address,
    earner_token_root: vector<u8>,
    token_indices: vector<u32>,
    token_tree_proofs: vector<vector<u8>>,
    token_leaf_tokens: vector<Object<Metadata>>,
    token_leaf_cummulative_earnings: vector<u64>,
  ) acquires RewardsConfigs
```

Called by the earner to claim his distributed rewards

`recipient`: the account that receives the rewards

`root_index`: the index of the distribution root

`earner_index, earner_tree_proof, earner, earner_token_root, token_indices, token_tree_proofs, token_leaf_tokens, token_leaf_cummulative_earnings`: specifies the earner, the distributed assets and respective amounts, as well as the merkle proofs showing that all of the provided data is included in the distribution Merkle tree.

### View functions

```move

public fun claimer_of(earner: address): address acquires EarnerStore

public fun cummulative_claimed(earner: address, token: Object<Metadata>): u64 acquires EarnerStore

public fun operator_token_shares(operator: address, token: Object<Metadata>): u128 acquires OperatorStore

public fun operator_shares(operator: address, tokens: vector<Object<Metadata>>): vector<u128> acquires OperatorStore

public fun get_withdrawability_and_scaling_factor_at_epoch

public fun can_withdraw(
  operator: address,
  token: Object<Metadata>,
  epoch: u64
): bool acquires OperatorSlashingStore

public fun share_scaling_factor(operator: address, token: Object<Metadata>): u64 acquires OperatorSlashingStore

public fun delegate_of(staker: address): address acquires StakerStore

public fun is_operator(operator: address): bool acquires StakerStore

public fun staker_token_shares(staker: address, token: Object<Metadata>): u128 acquires StakerStore

public fun staker_nonormalized_shares(staker: address): (vector<Object<Metadata>>, vector<u128>) acquires StakerStore

public fun total_shares(pool: Object<StakingPool>): u128 acquires StakingPool

public fun withdrawal_delay(tokens: vector<Object<Metadata>>): u64 acquires WithdrawalConfigs
```