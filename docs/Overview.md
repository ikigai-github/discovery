# Protocol Components

## Overview
- Discovery: Handles the logic for user commitments: accepting, modifying, consuming and distributing rewards.
- Liquidity
- Folds
  - Commit
  - Reward
  - Collect
  - Distribute
- Misc

## Discovery
 
### Stake Validator 
   - Unparameterized
   - Checks: 
      - Doesn't have FoldToken (appears to be commented out) 

### Discovery Validator
#### Parameters:
```
data PDiscoveryLaunchConfig (s :: S) 
= PDiscoveryLaunchConfig
  ( Term
      s
      ( PDataRecord
          '[ "discoveryDeadline" ':= PPOSIXTime
            , "penaltyAddress" ':= PAddress
            , "globalCred" ':= PStakingCredential
            ]
      )
  )
```
#### Redeemers:
```
data PNodeValidatorAction (s :: S)
= PLinkedListAct (Term s (PDataRecord '[]))
| PModifyCommitment (Term s (PDataRecord '[]))
| PRewardFoldAct (Term s (PDataRecord '[]))
```
- PLinkedListAct
  - Used for initial commit/decommit. Or other finset operations?
  - Relies on FinSet validation. Only checks that a NodeCS is minted/burned.
- PModifyCommitment
  - Updates an existing commitment
  - Checks that
    - the datum remains unchanged
    - only the ada value changes
    - ada value is not reduced
    - nothing is minted or burned
    - the deadline has not passed
- PRewardFoldAct
  - Used by fold operations (?)
  - Checks only that the user stake exists

### MP
#### Parameters:
```
data PDiscoveryConfig (s :: S)
  = PDiscoveryConfig
      ( Term
          s
          ( PDataRecord
              '[ "initUTxO" ':= PTxOutRef
               , "discoveryDeadline" ':= PPOSIXTime
               , "penaltyAddress" ':= PAddress
               ]
          )
      )
```
#### Redeemers:
```
data DiscoveryNodeAction
  = Init
  | Deinit
  | -- | first arg is the key to insert, second arg is the covering node
    Insert PubKeyHash DiscoverySetNode
  | -- | first arg is the key to remove, second arg is the covering node
    Remove PubKeyHash DiscoverySetNode
```
- Init
  - Used to create the head node, initializing the set
  - Checks: 
    - the transaction consumes the parameterized TxOutRef
    - `pInit` passes
      - tx does not spend nodes
      - tx outputs one node
      - mints the correct node token only
- Deinit
  - Removes a lone head node, fully ending an empty set
  - Checks (all contained in `pDeinit`):
    - tx spends exactly one node
    - tx doesn't output any nodes
    - burns the correct node token only
  - Should check that the reward fold has been completed, doesn't yet.
- Insert
  - Inserts a node in between two specified nodes
  - Checks:
    - The discoveryDeadline hasn't passed
    - The tx is signed by the owner of the new node
    - `pInsert` passes
      - covering node is covering
      - consumes exactly one node
      - covering node output includes correct value & datum
      - new node output is valid
      - mints the correct node token only
- Remove
  - Removes a node, ensuring its neighbors are linked by reference
  - Checks:
    - The deadline hasn't passed and `pClaim` passes
    
    OR
    - The deadline has passed and `pRemove` passes

#### Common.hs Utilities
- `makeCommon`
  - simple abstraction of boilerplate. parses inputs for all redeemers. 
  - checks all inputs from the NodeValidator are of the same origin
- `pInit`
  - additional logical checks for the Init redeemer (checks shown above)
- `pDeinit`
  - logical checks for the Deinit redeemer (checks shown above)
- `pInsert`
  - additional logical checks for the Insert redeemer (checks shown above)
- `pRemove`
  - additional logical checks for the Remove redeemer
  - Checks:
    - Covering key covers removed node
    - Spends exactly 2 nodes
    - Exactly one node is output
    - burns the correct node token only
    - signed by the node owner
    - withdrawal penalty is paid
- `pClaim`
  - additional logical checks for the Remove redeemer
  - Checks:
    - Removes correct node
    - burns the correct node token
    - signed by the node owner
    - the node has been processed by the rewards fold


## Liquidity
 - Stake Validator
 - Validator
 - MP

## Commit Fold
### Validator 
Defined by `pfoldValidatorW`

#### Redeemers
```
data PFoldAct (s :: S)
  = PFoldNodes
      ( Term
          s
          ( PDataRecord
              '[ "nodeIdxs" ':= PBuiltinList (PAsData PInteger) ]
          )
      )
  | PReclaim (Term s (PDataRecord '[]))
```
- PFoldNodes
  - Checks (via `pfoldNodes`):
    - has one input
    - does not mint or burn
    - the current node's key is output to the new node
    - the new node's `next` and  `committed` fields match the commit fold state (?)
    - the datum does not change owners
    - no assets (excluding ada) are taken from or added to the valdator address
    - the validity window falls before the discovery deadline
- PReclaim

### MP
Defined by `pmintFoldPolicyW`.
#### Redeemers
```
data PFoldMintAct (s :: S)
  = PMintFold (Term s (PDataRecord '[]))
  | PBurnFold (Term s (PDataRecord '[]))
```
- PMintFold
  - Checks (via `pmintCommitFold`):
    - exactly one token from this policyId is minted
    - one commit fold token is output
    - the output datum currNode points to the referenced datum
    - the output datum has no commitment
    - the reference input includes an origin node token
    - the validity window falls before the discovery deadline
- PBurnFold
  - Checks (via `pburnCommitFold`):
    - exactly one token from this policy is burned

## Reward Fold
 - Validator
 - MP

## Collect Fold
 - Validator
 - MP

## Distribute Fold
 - Validator
 - MP

## Misc
- Always Fails
- Proxy Token Holder v1
- Auth Mint