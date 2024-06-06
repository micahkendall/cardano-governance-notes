# Micah's Notes on Cardano Governance

I read the [cardano ledger specification](https://intersectmbo.github.io/formal-ledger-specifications/pdfs/cardano-ledger.pdf). This covers the technical functionality of governance. The governance system is a multi-stage process, with a constitutional committee, delegated representatives, and stake pool operators all having a say in the future of the Cardano protocol.

## Protocol:
The version is two natural numbers: N x N
The treasury & reserves are both ada amounts that are part of the core protocol.

Parameters are core settings of the entire Cardano protocol, these may be adjusted by proposals.

The parameters are split into 5 groups:
1. Network Group
2. Economic Group
3. Technical Group
4. Governance Group
5. *Security Group

Each parameter is in groups 1-4, and optionally a member of group 5. 
* Security group means that 50% of SPO stake must accept any proposal to change.

Here are the parameters:
### Network
The network parameters have 2nd order effects on things like finality and so shouldn't be changed lightly. Block size increases should also be reserved for critical features such as Leios.
1. maxBlockSize, a natural number indicating the maximum size in bytes of a block.
2. maxTxSize, a natural number indicating the maximum size in bytes of a transaction.
3. maxHeaderSize, a natural number indicating the maximum size in bytes of a block header.
4. maxValueSize, a natural number indicating the maximum size in bytes of a value.
5. maxCollateralInputs, a natural number indicating the maximum number of collateral inputs in a transaction.
6. maxTxExUnits, the maximum step & memory execution units that may be consumed in one tx.
7. maxBlockExUnits, the maximum step & memory execution units that may be consumed in one block.
### Economic Group
These influence the economic incentives of the Cardano system. Largely these minimise network spam.
1. Parameter a, "txFeePerByte", and parameter b, "txFeeFixed", are natural numbers that determine the fee for a transaction. The algorithm: `ceil(b + a*size(tx))` where size(tx) is the byte size of the cbor-serialised transaction. This is here because the bandwidth for transactions is a scarce public resource.
* fees paid from transactions go into the treasury.
2. KeyDeposit, an amount of lovelace  indicating the deposit locked to register a stake key and hence delegate stake. The delegation of keys to pools is a large mapping stored in the ledger state, and this deposit is to prevent spamming the ledger with keys.
3. PoolDeposit, an amount of lovelace indicating the deposit locked to register a stake pool. This exists for similar reasons to KeyDeposit.
4. CoinsPerUTxOByte, the amount of lovelace that must be in each UTxO entry, per byte of it's serialised cbor form. This is to prevent spamming the ledger with tiny UTxO entries.
5. MinFeeRefScriptCoinsPerByte, the cost per byte of serialised flat-encoded referenced scripts paid in a transaction. When you reference a script in a transaction, you are essentially retrieving deep storage and into active memory, this costs you for that.
6. Prices, fractional numbers indicating how much lovelace should be charged for memory and steps execution units. This is a simple 2 element array.
### Technical Group
1. a0, "PoolPledgeInfluence"
2. Emax, "RetireMaxEpoch"
3. nopt, "k", "PoolTargetNum", the target number of stake pools.
4. collateralPercentage, to indicate how much more collateral than fee must be paid in the event of transaction invalidity. This disincentivises abuse of the plutus machine evaluator, one of the most expensive parts of the ledger.
5. costMdls, "Cost Models", this is a complex set of pricing models for the execution of scripts. Whenever scripts are added to the ledger, they are executed by the ledger, and this costs computational resources. Different builtins are costed due to their resource usage. These aren't priced in lovelace, these step values are summed for each redeemer and used alongside the "Prices" parameter.
### Governance

1. Thresholds:
    - drepThresholds, the rational numbers P1,
P2a, P2b, P3, P4, P5a, P5b, P5c, P5d, and P6, 
    - and poolThresholds, the rational numbers Q1, Q2a, Q2b, Q4 and Q5e
    - These are the thresholds for the various voting stages in the governance process. The thresholds are rational numbers, and the voting process is a multi-stage process.
2. ccMinSize: the minimum size of the constitutional committee.
3. ccMaxTermLength: the maximum term length of the constitutional committee, in epochs. Epochs are approximately 5 days long.
4. govActionLifetime, how long before a governance action expires. 
5. govActionDeposit, the lovelace deposited when you put a proposal on the ledger. Similar to keyDeposit or poolDeposit, this is returned regardless of whether the proposal is accepted, rejected, or expires.
6. drepDeposit, the lovelace deposited to register as a delegate representative (delegate receiver). This is returned when you deregister. Similar to govActionDeposit.
7. drepActivity, how often a drep must vote to remain active, in epochs. A drep may either explicitly abstain, vote yes or no to stay active. Not immediately, but during a later state transition after voting, the dRep activity timer is set into the value-of-this-field epochs into the future.

A protocol parameters update is a change in any of the above.

## Governance:
There are three bodies in Cardano governance:
1. **CC:** The constitutional committee
2. **DRep:** The delegated represenatives
3. **SPOs:** Stake pool operators

A "GovRole" is either a CC, DRep, or SPO.

*Note: all three of these are democratically elected by ADA holders in some form.*

### Actions of Governance
1. A no confidence decision.
    - This is a vote of no confidence in the current CC.
2. Place a new CC.
    - The set of credentials and the activation epoch are specified in this action.
3. Place a new constitution
    - A document hash (social, not functional), and an optional script hash, are specified in this action.
4. Trigger a hard fork
    - the new protocol version is specified in the action.
5. Change the protocol parameters
    - a set of new values of the protocol parameters is specified in the action.
6. Withdraw from the treasury
    - the particular amount and reward address are specified in the action.
7. Info, a governance action that doesn't change the ledger state, but allows for social proposals.

Actions are proposed in transactions. When you propose an action, you must deposit a certain amount of lovelace. This is returned if the action is accepted, rejected, or expires. Multiple actions may be proposed in a single transaction, each has an index. Actions are identified by the transaction hash of the transaction that proposed them, and the index of the action in that transaction. The "GovActionID" is hence similar to the "OutputReference" of a UTxO entry.

Delegators are ada holders who have registered by placing their KeyDeposit. They may publish a delegation certificate which contains a "VDeleg" field, either pointing to a DRep's credential, abstaining, or indicating a vote of no confidence in the current CC. The delegation certificate is signed by the delegator's key. The vote is weighted by their ADA holdings under that key.

Each type of action, 1-7, has a hash protection because each action links to the previous state of what that action effects. No confidence and constitutional committee actions are linked together, all other actions are in their own indepedent state. An action will fail to be enacted if the previous state is not the expected state. Treasury withdrawals and the information action are both exempt and do not apply the strict protection rules.

### Governance Proposals
Proposals are the transactions where the governance actions are posted. For the transaction to be valid, the proposal rules must validate:
- A pointer must be provided to the previous action if under the "hash protection" rule.
- There must be a deposit placed, the "govActionDeposit" parameter specifies the lovelace amount.
- An anchor is posted, which is an off-chain document URL and Hash. This is to provide a human-readable explanation of the proposal.

In the Info proposal, there are no changes, however an anchor is attached. This allows you to make a plaintext or rich-document proposal, which is not enforced by the ledger.

### Voting
Voting is done in transactions. Multiple votes may be in one transaction. The vote has three options: yes,no,abstain. Votes contain the "GovActionID", pointing to the particular governance proposal being voted on, and also contain the "Voter", the credential and role. An anchor (URL + Hash) may be attached, for the purpose of explaining why a vote was cast.

CC members automatically abstain if they have failed to register a hot key, they have expired, or resigned (deregistered hot key). Members of the CC always have equal voting power.

Active DReps who don't vote, automatically vote no. Inactive dreps automatically abstain.

The default voting status of SPOs depends on the action.

One interesting implication of the Voter being the combination of credential and role is that the same credential may vote seperately under different roles. A constitutional commitee member who is also registered as a DRep might personally vote in favour, under their DRep role, but against under their CC role.

### Governance Transactions
The relevant transaction fields to governance are:
- "txvote", a list of votes
- "txprop", a list of proposals
- "txcerts", a list of certificates

All votes happen in the transaction transition machine, and all proposals are put forward as well. Enactment is external to transactions and has it's own state flow.

Transaction certificates (txcerts) are relevant for the registration of DReps & Constitutional Committee members, and the delegation process.

### Governance Certificates
1. Delegation
    - delegation is important and allows you to put your ADA towards a delegated represenative who may vote on your behalf. This is the implementation of representative democracy in Cardano.
    - SPOs are also delegated to in this manner, and as they have a voting say, has similar functionality.
2. Register DRep
    - add a drep to the ledger state.
    - resets the drep activity period.
3. CC Register Hot
    - Anyone may register a "constitutional committee hot key", the key that is used to actively vote on the constitution, under a cold key. However it is not active meaningfully unless the corresponding cold key is actually included in the constitutional commitee key set. This functionality is to prevent delays, so an upcoming CC member may register their hot key in advance.

## Enactment, Ratification
Actions are ratified through on-chain voting. At least 2 of the 3 bodies are engaged for all votes.

The no-confidence action, the new constitutional committee action, a constitution change, and the hard fork, all delay ratification until the first epoch after their enactment. This gives the CC time to vote on proposals, reevaluate how to interpret a new constitution, and how to act under a new hard-fork.

Thresholds for enactment:
1. NoConfidence
    - CC is ignored
    - DRep threshold is P1
    - SPO threshold is Q1
2. NewCC
    - if the CC is confident, the dRep threshold is P2a, and the SPO threshold is Q2a
    - if no confidence is expressed, the dRep threshold is P2b, and the SPO threshold is Q2b
3. NewConstitution
    - CC must approve
    - DRep threshold is P3
    - SPOs get no vote
4. HardFork
    - CC must approve
    - DRep threshold is P4
    - SPOs threshold is Q4
5. ChangeParameters
    - CC must approve
    - DRep threshold is P5, a-d depending on the parameter groups, the maximum is taken
    - If any of the parameters are in the security group, the SPOs threshold is Q5e, otherwise SPOs can be ignored
6. TreasuryWithdrawal
    - CC must approve
    - DRep threshold is P6
    - SPOs are ignored
7. Info
    - approval is not relevant because this is never enacted, the reached thresholds are merely indicative of the community's opinion.

Ratification may accept, reject, or continue, so failure to meet threshold immediately does not necessarily end the proposal. The govActionLifetime parameter indicates how long an action may continue for.

to-be-enacted proposals are rejected if:
- NewCommittee case, the maximum term length has not been reached yet.
- TreasuryWithdrawal, the treasury has insufficient funds.

**Unanswered Questions for the more knowledgeable experts than me:**
- How is the threshold for the CC defined? Is it a simple majority or some supermajority?
- What threshold is considered for the no-confidence action?
- How long does a no-confidence vote take to enact?
- Generally how long do votes take to enact?
- Can you provide detailed explanation of the delayingAction function?
