---
CIP: 1694
Title: An On-Chain Decentralized Governance Mechanism for Voltaire
Status: Proposed
Category: Ledger
Authors:
    - Jared Corduan <jared.corduan@iohk.io>
    - Matthias Benkort <matthias.benkort@cardanofoundation.org>
    - Kevin Hammond <kevin.hammond@iohk.io>
    - Charles Hoskinson <charles.hoskinson@iohk.io>
    - Samuel Leathers <samuel.leathers@iohk.io>
Discussions:
    - https://github.com/cardano-foundation/cips/pulls/?
Created: 2022-11-18
License: CC-BY-4.0
---

# An On-Chain Decentralized Governance Mechanism for Voltaire

## Abstract

We propose a revision of Cardano's on-chain governance system to support the new requirements for Voltaire. The existing specialized governance support for protocol parameter updates and MIR certificates will be deprecated, and two new fields will be added to normal transaction bodies:

1. governance actions;
2. votes.

**Any Cardano user** will be allowed to submit a **governance action**. Three distinct groups will be responsible for ratifying these governance actions using their **votes**:

1. a Constitutional committee;
2. a group of delegation representatives (henceforth called **DReps**); and
3. the stake pool operators (henceforth called **SPOs**).

Ratified actions may then be **enacted** on-chain, following a set of well-defined rules.

As with stake pools, any Ada holder may register to be a DRep and so
choose to represent themselves if they wish, or they may, instead, delegate their voting rights
to any other registered representative.  These voting rights will be based on Ada holdings.

> **Acknowledgements:**
> Many people have commented on and contributed to this document. We would especially like to thank the following people for providing their wisdom and insights:
>
> * Jack Briggs;
> * Tim Harrison;
> * Andre Knispel;
> * Philip Lazos;
> * Michael Madoff;
> * Evangelos Markakis;
> * Joel Telpner;
> * Thomas Upfield.

## Motivation

We're heading into the age of Voltaire, laying down the foundations for decentralized decision-making. This CIP describes a mechanism for on-chain governance that will underpin the Voltaire phase of Cardano.
The document builds on and extends the original Cardano governance scheme that was based on a fixed number of governance keys.
It aims to provide a first step that is both valuable and technically achievable in the near term as part of the proposed Voltaire governance system.

It also seeks to act as a jumping-off point for continuing community input, including on appropriate threshold settings and other on-chain settings.
Subsequent proposals may adapt and extend this proposal to meet emerging governance needs.

### Current Design

The existing on-chain Cardano governance mechanism (introduced in the Shelley ledger era) is capable of:

1. modifying the values of the protocol parameters (including initiating "hard forks"); and
2. transferring Ada out of the reserves and the treasury (and also moving Ada between the reserves and the treasury).

In the current scheme, governance actions are initiated by special transactions that require `Quorum-Many` authorizations from the governance keys (5 out of 7 on the Cardano mainnet)[^1].
Fields in the transaction body provide details of the action to be carried out: whether changing protocol parameter changes or initiating funds transfers. Each transaction can trigger precisely one kind of action, but a single action can make multiple changes.

- Protocol parameter updates use [transaction field nº6](https://github.com/input-output-hk/cardano-ledger/blob/8884d921c8c3c6e216a659fca46caf729282058b/eras/babbage/test-suite/cddl-files/babbage.cddl#L56) of the transaction body.
- Movements of the treasury and the reserves use [Move Instantaneous Rewards (abbrev. MIR) certificates](https://github.com/input-output-hk/cardano-ledger/blob/8884d921c8c3c6e216a659fca46caf729282058b/eras/babbage/test-suite/cddl-files/babbage.cddl#L180).

Successful governance actions are applied on an epoch boundary (they are **enacted**).

One of the protocol parameters is sufficiently significant to merit special attention: changing the major protocol version enables Cardano to enact controlled hard forks. This type of update, therefore, has a special status among the possible protocol parameter updates.

### Shortcomings of the Shelley Governance Design

The current design was intended to provide a simple, transitional approach to governance.
This proposal aims to address a number of shortcomings with that design.

1. It gives no room for active on-chain participation of Ada holders. While changes to the protocol are usually the results of discussions with selected community actors, the process is driven mainly by the founding entities.
Ensuring everyone can voice their concern is cumbersome and can be perceived as arbitrary at times.

2. Movements from the treasury can be hard to track and constitute a critical and sensitive topic.
It is important to have more transparency and more layers of control over these movements.

3. While they need to be treated specially by SPOs, hard forks are not differentiated from other protocol parameter changes.

4. Finally, while there's currently a somewhat common vision for _Cardano_ that is shared by its founding entities and many community members, there is no clearly defined document where these guiding principles are recorded. It makes sense to leverage the Cardano blockchain to record the ethos of the project itself in an immutable fashion, as the formal Cardano Constitution.

## Specification

+ [The Cardano Constitution](#the-constitution)
+ [The Constitutional Committee](#the-constitutional-committee)
  - [Initial Committee](#initial-committee)
  - [Replacing The Committee](#replacing-the-committee)
+ [Governance Actions](#governance-actions)
  - [Ratification](#ratification)
    * [Requirements](#requirements)
    * [Lifecycle](#lifecycle)
  - [Enactment](#enactment)
  - [Content](#content)
  - [Governance Action IDs](#governance-action-ids)
+ [Votes](#votes)
  - [Governance State](#governance-state)
  - [Stale Votes](#stale-votes)
+ [Delegated Representatives (DReps)](#delegated-representatives--dreps-)
  - [New Stake Distribution for DReps](#new-stake-distribution-for-dreps)
  - [Definitions Surrounding Voting Stake](#definitions-surrounding-voting-stake)

### The Cardano Constitution

The Constitution is a text document that defines Cardano's shared values and guiding principles. At this stage, it is meant to be an informational document that unambiguously captures the Cardano core values. At a later stage, we can imagine the Constitution perhaps evolving into a smart-contract-based set of rules driving the entire governance framework. For now, however, the Constitution will remain an off-chain document whose hash digest value will be recorded on-chain.

### The Constitutional Committee

We define a _Constitutional Committee_ which represents a set of individuals or entities (associated with a pair of Ed25519 credentials) who are responsible for overseeing the governance actions that are defined in the section below and ensuring that the Constitution is respected.

The Constitutional Committee is considered to be in one of the following two states at all times:

1. a normal state (i.e. one of confidence); or
2. a state of no-confidence

In a _state of no-confidence_, the current committee is no longer able to participate in governance actions
and must be replaced before any governance actions can be enacted (see below).  Any outstanding governance actions become void immediately the committee
enters a state of no-confidence.

The Constitutional Committee will use a hot and cold key setup. Hot keys will re-use the existing "genesis delegation certificate" mechanism that has been in place since the start of the Shelley era.

#### Initial Constitutional Committee

The initial Constitutional Committee will constitute the core members of a member-based organization that is dedicated to the development of Cardano. The final list of members is yet to be defined. However, it will, in all likelihood, be made of some of the founding entities, such as Input Output Global and the Cardano Foundation, as well as key community actors who are interested in participating in the Cardano governance process.

#### Replacing the Constitutional Committee

The Constitutional Committee can be replaced in one of two ways:

* When in a **normal** state (i.e. a state of **confidence**), the committee can be replaced via a specific governance action (action 2 below) that requires the approval of both the **current Constitutional Committee** and the **DReps**.

* When in a state of **no-confidence**, the committee can also be replaced via a specific governance action (action 5 below), but this requires the approval of the **SPOs** and the **DReps** instead.

#### Size of the Constitutional Committee

Unlike the Shelley governance design, the size of the Constitutional Committee is not fixed.  It may be changed any time that a new committee is installed.  Likewise, the _quorum_ (the number of votes that are required to enact governance actions) is not fixed and can be varied whenever a new committee is installed. This gives a great deal of flexibility.

### Governance Actions

We define six different types of **governance actions**. A governance action is an on-chain event that is triggered by a transaction and has a deadline after which it cannot be enacted.

An action is said to be **ratified** when it gathers enough votes in its favor (through rules and parameters detailed below). An action that doesn't collect sufficient `yes` votes before its deadline is said to have **expired**.
An action that has been ratified is said to be **enacted** once it has been activated on the network.
Regardless of whether they have been ratified, actions may, however, be **dropped** without being **enacted** if, for example, a motion of no confidence is enacted.


| Action                                         | Description                                                                                        |
| ---                                            | ---                                                                                                |
| 1. Motion of no-confidence                     | A motion to create a _state of no-confidence_ in the current Constitutional Committee  |
| 2. New Constitutional Committee and/or quorum size | Changes to the members of the Constitutional Committee and/or to its signature threshold   |
| 3. Updates to the Constitution                 | A modification to the off-chain Constitution, recorded as an on-chain hash of the text document                          |
| 4. Hard-Fork[^2] Initiation                    | Triggers a non-backwards compatible upgrade of the network; requires a prior software upgrade |
| 5. Protocol Parameter Changes                  | Any change to one or more updatable protocol parameters, excluding changes to major protocol versions ("hard forks")                                  |
| 6. Treasury Withdrawals                        | Movements from the treasury, sub-categorized into small, medium or large withdrawals (based on the amount of Lovelace to be withdrawn). The thresholds for treasury withdrawals are discussed below.             |

Any Ada holder can submit a governance action to the chain. They must provide a deposit of `Gov-Deposit` Lovelace, which will be returned when the action is finalized (whether it is **ratified**, has been **dropped**, or has **expired**).

Note that a motion of no-confidence is an extreme measure that enables Ada holders to revoke the power that has been granted to the current Constitutional Committee. Any outstanding governance actions, including ones that the committee has ratified, will be dropped if the motion is enacted. As for other governance actions, votes ensue, followed by ratification or expiry.

#### Ratification

Governance actions are **ratified** through on-chain voting actions.  Different kinds of governance actions have different ratification requirements: depending on the type of governance action, an action will become ratified if some specific combination of the following occurs:

* the Constitutional Committee approves of the action (`Quorum-Many` members vote 'yes');
* the DReps approve of the action (the stake controlled by the DReps who vote 'yes' meets a certain threshold of the total registered voting stake);
* the SPOs approve of the action (the stake controlled by the SPOs who vote 'yes' meets a certain threshold over the total registered voting stake).

> **Warning**
> As explained below, different stake distributions apply to DReps and SPOs.

##### Requirements

The following table details the ratification requirements for each governance action scenario. The columns represent:

* **Governance Action Type**<br/>
  The type of governance action. Note that the three possible treasury actions involve Lovelace amounts $T_0$, $T_1$, $T_2$, and $T_3$.

* **Constitutional Committee**<br/>
  A value of :heavy_check_mark: indicates that `Quorum-Many` Constitutional Committee 'yes' votes are required.<br/>
  A value of :x: means that Constitutional Committee votes do not apply.

* **DReps**<br/>
  The DRep vote threshold that must be met as a percentage of *active voting stake*, ranging from 0 to 100 (inclusive).

* **AVST**<br/>
  The _**A**ctive **V**oting **S**take **T**hreshold_. The percentage to be used to determine if there is *sufficient active voting stake*.

* **AVST fallback**<br/>
  The fallback condition if the AVST threshold is not met
  * **None**: there is no fallback, the action cannot be ratified unless the AVST threshold is met
  * **SPO vote**: the action can be ratified if there is a sufficient SPO vote endorsement.


* **SPOs**<br/>
  The SPO vote threshold which must be met as a percentage of the stake held by all stake pools. The SPO vote is only considered if the AVST threshold is :x: or the AVST is below the AVST threshold.

| Governance Action Type                               | Constitutional Committee | DReps    | AVST     | AVST Fallback | SPOs     |
| ---                                                  | :---:                    | ---      | ---      | ---           | ---      |
| 1. Motion of no-confidence                           | :x:                      | $P_1$    | $Q_1$    | None          | $R_1$    |
| 2(a). New Committee/quorum (_normal state_)          | :heavy_check_mark:       | $P_{2a}$ | $Q_{2a}$ | SPO Vote      | $R_{2a}$ |
| 2(b). New Committee/quorum (_state of no-confidence_)| :x:                      | $P_{2b}$ | $Q_{2b}$ | None          | $R_{2b}$ |
| 3. Update to the Constitution                        | :heavy_check_mark:       | $P_3$    | $Q_3$    | SPO Vote      | $R_3$    |
| 4. Hard-Fork initiation                              | :heavy_check_mark:       | $P_4$    | 0        | -             | $R_4$    |
| 5. Protocol parameter changes                        | :heavy_check_mark:       | $P_5$    | $Q_5$    | SPO Vote      | $R_5$    |
| 6(a). Treasury withdrawal, $[T_0, T_1)$              | :heavy_check_mark:       | $P_{6a}$ | $Q_{6a}$ | SPO Vote      | $R_{6a}$ |
| 6(b). Treasury withdrawal, $[T_1, T_2)$              | :heavy_check_mark:       | $P_{6b}$ | $Q_{6b}$ | SPO Vote      | $R_{6b}$ |
| 6(c). Treasury withdrawal, $[T_2, T_3)$              | :heavy_check_mark:       | $P_{6c}$ | $Q_{6c}$ | SPO Vote      | $R_{6c}$ |

Some of the parameters given in this table ( $P_1$ ... $R_{6c}$, $T_1$ ... $T_3$ ) may be updatable protocol parameters, but others should be hard-coded. This proposal deliberately leaves both this choice and the choice of actual parameter values open for discussion.

> **Note**
> For all treasury withdrawals, the withdrawal threshold is the **total** amount of Lovelace that is withdrawn by the action, not the amount of any single withdrawal if the action specifies more than one withdrawal.

##### Governance Action Lifecycle

Governance actions are checked for ratification only on an epoch boundary. This delay allows everyone to vote on each proposal and prove that they are active.

**At most one** governance action **of each type** (i.e. one from each of the six categories listed above) can be staged for enactment in any given epoch. For example, there can only be one treasury withdrawal action in a single epoch (it may, however, comprise many individual withdrawals).

Once ratified, actions will be staged for enactment. Actions that have been staged will be enacted on the following epoch boundary, unless they are dropped. All submitted governance actions will therefore either:

1. be **ratified**;
2. be **dropped** as a result of some higher priority action; or else
3. will **expire** after a number of epochs.

Deposits are returned immediately when either:

1. a ratified action is **enacted**;
1. the action **expires**; or
1. a ratified action is **dropped**.

Ratification is explained in more detail later in this document.

>**Note:**
>This design means that the first governance action of a given type to be ratified will be staged for enactment in an epoch.

#### Enactment

Ratified actions may be enacted at an epoch boundary. During enactment, actions in the staging group for the current epoch are prioritized as follows:

1. Motion of no-confidence;
2. New Constitutional Committee or a change to the quorum size;
3. Updates to the Constitution;
4. Hard Fork initiation;
5. Protocol parameter changes;
6. Treasury withdrawals.

As described above, at most one action of each type may be enacted in any given epoch.
So a ratified _"Motion of no-confidence"_ action will be enacted first if there is one, and so on.

A successful _"Motion of no-confidence"_, the election of a new Constitutional Committee, or a constitutional change invalidates *all* other **unenacted** governance actions (whether or not they have been ratified),
_causing them to be immediately **dropped** without ever being enacted_. Deposits for dropped actions will be returned immediately.

#### Content of a Governance Action

Every governance action will include the following:

* a deposit amount (recorded since the amount is an updatable protocol parameter);
* a reward address to receive the repaid deposit;
* a URL to any metadata that is needed to justify the action;
* a hash of the contents of this metadata URL.

// TODO: Provide a CBOR specification in the annexe for this new on-chain entity

Additionally, the action will include the following:

1. For protocol parameter changes - the changed parameters;
2. For hard fork initiation - the new major protocol version, which must be one greater than the current version;
3. For treasury withdrawals - a map from stake credentials to a positive number of Lovelace;
4. For updates to the Constitution - a 32-byte blake2b-256 hash digest of the Constitution document;
5. For new Constitutional Committee and changes to the quorum size - a set of key hashes and a positive number that is no greater than the size of the committee;
6. For a motion of no confidence - nothing else.

The deposit amount will be added to the _deposit pot_, similar to stake key deposits.

#### Governance Action IDs

Each accepted governance action will be assigned a unique identifier (a.k.a. the **governance action ID**), consisting of the transaction ID that created it and the index within the transaction body that points to it.

### Votes

Each vote transaction consists of the following:

* a governance action ID;
* a role - Constitutional Committee, DRep, or SPO;
* a key-hash (which will be verified to have the role above);
* a URL for any metadata that is relevant to the vote;
* a hash of the contents of this URL;
* a yes/no/abstain vote.

Note that "abstained" votes are not included in the "active voting stake".
Note also that a vote to abstain is different than abstaining from voting
(the former is visible on chain, the latter is not).
To avoid confusion, we will only use the word "abstain" from this point onward to mean a vote
to abstain.

The key hash will trigger the appropriate signature check on the transaction body according to the existing `UTxOW` ledger rule.

Votes can be cast multiple times for each governance action by a single key hash. Correctly submitted votes override any older votes for the same key hash and role.
As soon as a governance action is ratified, voting ends. Any future votes are not considered or recorded (whether they are yes, no or abstain).

#### Governance State

When a governance action is successfully submitted to the chain, its progress will be tracked by the ledger state.
In particular, the following will be tracked:

* the governance action ID;
* the epoch that the action expires;
* the deposit amount;
* the rewards address that will receive the deposit when it is returned;
* the total yes/no/abstain votes of the Constitutional Committee for this action;
* the total yes/no/abstain votes of the DReps for this action;
* the total yes/no/abstain votes of the SPOs  for this action.

#### Stale Votes

The votes from the DReps and the SPOs may become meaningless after crossing an epoch boundary since they may become unregistered. Therefore, all unregistered votes are cancelled before any new votes are considered.

#### Changes to the Stake Snapshot

Since the stake snapshot changes at each epoch boundary, a new current voting tally must be calculated for each governance action based on the votes that have been cast before any new votes are counted.  This avoids the same stake being used two or more times.  _This new tally may result in immediate ratification of a governance action._

### Delegated Representatives (DReps)

> **Warning**
> The design of the DReps is still subject to change and should not be conflated with Project Catalyst DReps.

Existing stake keys will be given new responsibilities. They will be able to delegate their stake to DReps. DRep Registration will mimic the existing stake delegation mechanisms (via certificates).

The following new types of voting certificates will be added:

* DRep registration certificate. Including:
  * DRep ID: hash of verification key
  * a URL for any metadata that is relevant to the DRep
  * a hash of the contents of this URL

* DRep retirement certificate. Including:
  * DRep ID
  * epoch number after which the DRep will retire

* Vote delegation certificate. Including:
  * stake credential
  * DRep ID

// TODO: Provide CBOR specification in the annexe for those new certificates.

The authorization scheme (i.e. which signatures are required) mimics the existing stake delegation certificate authorization scheme.

#### New Stake Distribution for DReps

In addition to the existing per-stake-credential distribution and the per-stake-pool distribution, we will now have a new per-DRep stake distribution.

This distribution will determine how much stake is backed by each `yes` vote from a DRep. The exact stake snapshot used for block production by the SPOs will also be used for the DReps for voting.

#### Definitions surrounding Voting Stake

We define some notions around voting stake:

* Lovelace contained in a transaction output is considered **active for voting** if:
  * It contains a registered stake credential.
  * The stake credential has delegated its voting rights to a registered DRep.
* Relative to some given percentage `P`, we say that there is **sufficient active voting stake** if
  the ratio of the *total active voting stake* to the *total stake in circulation* is at least `P`.
* Relative to a percentage `P`, a DRep (SPO) **vote threshold has been met** if the sum of the relative stake controlled by the DReps (SPOs) voting 'yes' to a governance action minus the sum of the relative stake controlled by the DReps (SPOs) voting 'no' is at least `P`.

> **Note**
> There are several alternative definitions for "the total stake in circulation":
>  1. The sum of the UTxO and the rewards accounts.
>  2. The sum of the UTxO, the rewards accounts, the fee pot, and the deposit pot.
>  3. The total ADA supply (i.e. 45 billion ADA) minus the reserves[^3].
>  We leave the choice open for discussion.

#### New Protocol Parameters

New protocol parameters will be needed for the following:

* the governance action deposit amount
* governance action expiration (as a number of epochs from the current epoch)
* thresholds for treasury withdrawals
* each of the ratification thresholds

As described above, some  of these will be updatable; others will be hard coded.

// TODO: Decide on the initial parameter values and whether they should be updatable.

// TODO: Decide on coherence conditions on the voting thresholds. For example, the threshold for a motion of no-confidence should arguably be higher than that of a minor treasury withdrawal.

In addition, the initial Constitutional Committee and the initial Constitution will need to be defined as part of a bootstrap process.

#### Changes to the existing ledger rules

* The `PPUP` transition rule will be rewritten and moved out of the `UTxO` rule and into the `LEDGER` rule as a new `TALLY` rule.

  It will process the governance actions and the votes, ratify them, and stage governance actions for enactment in the current or next epoch, as appropriate.

* The `NEWEPOCH` transition rule will be modified.
* The `MIR` sub-rule will be removed.
* A new `ENACTMENT` rule will be called immediately after the `EPOCH` rule. This rule will enact governance actions that have previously been ratified.
* The `EPOCH` rule will no longer call the `NEWPP` sub-rule or compute whether the quorum is met on the PPUP state.

#### More on DRep incentives

The DReps arguably need to be compensated for their work.

The corresponding incentive mechanisms need to be specified, with the funds probably coming from the per-epoch treasury allocation.
Performance constraints will also need to be considered since it would be problematic if millions of DReps were expected to vote on each governance action. Some incentive options to ensure a manageable number of DReps include:

* Requiring a large deposit when registering oneself as a DRep.
* An incentive scheme similar to the stake pool reward scheme is used to limit individual rewards. (Note that this is probably not enough on its own, as many voters may wish to be their own DRep regardless of payment).
* Some form of a per-epoch randomized sub-committee of DReps which takes stake weight into account.


## Rationale

### Role of the Constitutional Committee

At first, the Constitutional Committee may sound like a special committee that's granted extra power over DReps. However, because DReps can replace a committee at any time and DReps votes are also required to ratify every governance action, the Constitutional Committee has no more (and may, in fact, have less) power than the DReps. Given this, what role does the committee play, and why isn't it superfluous? The answer is that the committee solves the bootstrapping problem of this new governance framework. Indeed, as soon as we pull the trigger and enable such a framework to become active on-chain, without a Constitutional Committee, there would rapidly need to be sufficient DReps, so that the system did not rely solely on SPO votes. We cannot yet predict how active the community will be in registering as DReps, nor how reactive other Ada holders will be regarding delegation of votes.

Thus, the Constitutional Committee comes into play to make sure that the system can transition from its current state into fully decentralized governance in due course. Furthermore, in the long run, the committee can play a mentoring and advisory role in the governance decisions by being a set of elected representatives who are put under the spotlight for their judgment and guidance in governance decisions.
Above all, the committee is required at all times to adhere to the Constitution and to ratify proposals in accordance with the provisions of the Constitution.

// TODO Should we force a re-election of the initial committee? Should every committee have a term limit?

### Intentional Omission of Identity Validation

Note that this CIP does not mention any kind of identity validation or verification for the members of the Constitutional Committee or the DReps.

This is intentional.

We hope that the community will strongly consider only voting for and delegating to those who provide something like a DID to make themselves known.
However, enforcing identity verification is very difficult without some centralized oracle, which we consider to be a step in the wrong direction.

### Piggybacking on stake pool stake distribution

The Cardano protocol is based on a Proof-of-Stake consensus mechanism, so using a stake-based governance approach is sensible. However, there are many ways by which one could define how to record the stake distribution between participants. As a reminder, network addresses can currently contain two sets of credentials: one to identify who can unlock funds at an address (a.k.a. payment credentials) and one that can be delegated to a stake pool (a.k.a. delegation credentials).

Rather than defining a third set of credentials, we instead propose to re-use the existing delegation credentials, using a new on-chain certificate to determine the governance stake distribution. This implies that the DReps can (and likely will) differ from SPOs, so creating balance. On the flip side, it means that the governance stake distribution suffers from the same shortcomings as that for block production: for example, wallet software providers must support multi-delegation schemes and must facilitate the partitioning of stake into sub-accounts should an Ada holder desire to delegate to multiple DReps.

However, this choice also limits the implementation effort for wallet providers down the line and minimizes the effort needed for end-users to participate in the governance protocol. The latter is a sufficiently significant concern to justify the decision. By piggybacking on the existing structure, we keep the system familiar to users and reasonably easy to set up in order to maximize the chance of success and the participation rate in the governance framework.

### Separation of Hard Fork Initiation from Standard Protocol Parameter Changes

In contrast to other protocol parameter updates, hard forks (or, currently, changes to the protocol's major version) require much more attention.
Indeed, while other protocol parameter changes can be performed without significant software changes,
a hard fork assumes that a super majority of the network has upgraded the node to support the new set of features that are introduced by the upgrade. This means that the timing of a hard fork event must be communicated well ahead of time to all Cardano users, and requires coordination between stake pool operators, wallet providers, DApp developers, and the node release team.

Hence, this proposal promotes hard fork initiations as a standalone governance action, separate from protocol parameter updates, as in the Shelley scheme.

### Treasury Withdrawals vs Project Catalyst

Project Catalyst is currently one of the main drivers for withdrawals from the Cardano treasury. Each Catalyst round is usually followed by hundreds - if not thousands - of MIR requests to deliver funding to selected projects. In this new governance framework, however, we will only permit one treasury withdrawal per epoch.

Since all withdrawal requests from the treasury must fit into a single transaction, this limits the number of projects that can be funded in a single epoch. If necessary, this can be worked around by e.g. splitting funding over multiple epochs, by transferring funds to a temporary holding pot, or by restricting the number of projects that are funded in each round.

### The purpose of the DReps

Nothing in this proposal limits the SPOs from becoming DReps.
Why do we have DReps at all?
SPOs are chosen purely for block production.
Voters can choose to delegate their vote to DReps without needing to consider whether they are
also a good block producer.

### AVST (action voting stake threshold)

The AVST exists to ensure that the votes of the DReps are meaningful.
For example, if only 10 Lovelace were properly delegated to DReps, and 9 Lovelace "vote yes" to
a governance action, we would doubt the legitimacy of the result since there are
billions of ADA in circulation.
A high enough AVST legitimizes the representation.

### Ratification Requirements Table

The requirements in the [ratification requirement table](#requirements) are explained here.
Most of the governance actions have the same kind of requirements:
the constitutional committee must reach quorum and the DReps must reach a sufficient number of
'yes' votes, except in the situation where the AVST is too low, in which case the SPOs vote in
place of the DReps.
This includes these actions:
* New Committee/quorum (normal state)
* Update to the Constitution
* Protocol parameter changes
* Treasury withdrawal

#### Motion of no-confidence

A motion of no-confidence represents the lack of confidence by the Cardano community in the
current Constitutional Committee, and hence the Constitutional Committee should have no
say whatsoever in this governance action.
In this situation, the SPOs and the DReps are left to represent the will of the community.
To prevent an under-representation of the DReps, the AVST (active voting stake threshold)
becomes a requirement for ratification.

// TODO We need to fully understand the trade-offs between using the DReps votes with and without
the AVST.

#### New Committee/quorum (state of no-confidence)

Similar to the motion of no-confidence, electing a constitutional committee
depends on both the SPOs and the DReps to represent the will of the community.
To prevent an under-representation of the DReps, the AVST (active voting stake threshold)
becomes a requirement for ratification.

// TODO We need to fully understand the trade-offs between using the DReps votes with and without
the AVST.

#### Hard-Fork initiation

Regardless of any governance mechanism, SPO participation is needed for any hard fork since it is
they who must upgrade their software.
For this reason, we make their cooperation explicit in the hard fork initiation governance action,
by always requiring their vote.
The Constitutional Committee also votes, signaling the constitutionality of a hard fork.
The DReps also vote, to represent the will of every stake holder, though the AVST is not required.
Requiring the AVST in this situation could block a critical upgrade,
though we may wish to reconsider this in the future.

// TODO We need to fully understand the trade-offs between using the DReps votes with and without
the AVST.

### New Metadata structures

Both the governance actions and the votes use new metadata fields,
in the form of URLs and integrity hashes
(mirroring the metadata structure for stake pool registration).
The metadata is used to provide context.
For example, votes by the constitutional committee need an explanation as to why it is in line with
the constitution.
Governance actions need to explain why the action is needed,
what experts were consulted, etc.
We do not want transaction size constraints to limit this explanatory data,
so we use URLs.

This does, however, introduce new problems.
If a URL does not resolve, what should be the expectation for voting on that action?
Should we expect everyone to vote 'abstain'?.
Is this an attack vector against the governance system?
In such a scenario, the hash pre-image could be communicated in other ways, but we should be
prepared for the situation.
Should there be a summary of the justification on chain?

##### Alternative: Use of transaction metadata

Instead of specific dedicated fields in the transaction format, we could instead use the existing transaction metadata field.

Governance-related metadata can be clearly identified by registering a CIP-10 metadata label.
Within that, the structure of the metadata can be determined by this CIP (exact format TBD), using an index to map the vote or proposal id to the corresponding metadata URL and hash.

This avoids the need to add additional fields to the transaction body, at the risk of making it easier for submitters to ignore.
However, since the required metadata can be empty (or point to a non-resolving URL), it is already easy for submitters to not provide metadata, and so it is unclear whether this makes the situation worse.

Note that transaction metadata is never stored in the ledger state, so it would be up to clients
to pair the metadata with the actions and votes in this alternative, and would not be available
as a ledger state query.


### Three levels of treasury withdrawals

Three different treasury withdrawal governance actions were introduced so that higher
withdrawals could have higher ratification thresholds.
It should be more difficult to ratify withdrawals for larger amounts.

Alternatively, we could introduce a single action for all treasury withdrawals,
and specify an increasing function from ADA to the thresholds.

Additionally, since the treasury allocation for each epoch is given in terms of a percentage of the
reward pot, we should also consider treasury allotment (the protocol parameter $\tau$)
when computing the thresholds.
For example, the treasury withdrawals for a given epoch could be limited roughly equal to the
amount of lovelace moved to the treasury from the reserve pot that epoch.

// TODO Specify a function which takes the treasury allotment and the treasury withdrawal amount
and returns the AVST, the DRep voting threshold, and the SPO voting threshold.

### Out of scope

The follow topics are considered out of scope for this proposal.

#### The contents of the constitution

The contents of the initial constitution are extremely important, as are any processes around amending it.
This is, however, separate enough from the on-chain protocol proposed in this CIP
to merit a separate and focused discussion.

#### Legal issues

Any potential legal enforcement of either the Cardano protocol or the Cardano constitution are
completely out of scope for this CIP.

#### Off chain standards for creating governance actions

The Cardano community should think deeply about standards and processes for handling the creation
of the governance actions specified in this CIP.
These standards, however, are outside the scope of this CIP.
In particular, the role of Project Catalyst in creating treasury withdrawal actions is outside the
scope of this CIP, except in cases where clarifying the distinction is helpful.

#### Private entities

How any private companies or individuals choose to delegate their ADA, either to stake pools
or to DReps, is outside the scope of this CIP (and arguably to the CIP process in general).

## Path to Active

### Acceptance Criteria

- [ ] A new ledger era is enabled on the Cardano mainnet, which implements the above specification. This will be split into two stages, as described above.

#### Other potential acceptance criterion

- [ ] Exercising all the governance actions on a public testnet, with sufficient participation.
- [ ] Having the constitution text approved by some process.
- [ ] Having the initial constitutional committee approved by some process.

### Implementation Plan

The features in this CIP require a hard fork.

This document describes an ambitious change to Cardano governance.
A two-stage implementation process will allow it to be rolled out more quickly, with some of the more complicated aspects
(DReps and constitutional changes) rolled out in a second stage.
In particular, the changes will be implemented with two hard forks.
This will also allow time
for fuller evaluation of/consultation on incentives and other significant issues,
while establishing the basis for governance in Voltaire.
Below is a high-level summary of an implementation plan for (very roughly) half of the plan.

1. Add a subset of (important) governance actions to the transaction body (while deprecating protocol parameter updates and MIR certificates).
  The following governance actions will be supported initially:
  * protocol parameters
  * hard forks
  * withdrawals from the treasury
  * Constitutional Committee changes
2. Add votes to the transaction body, but disallow DRep votes.
3. The first version of the new `RATIFY` rule will only count Constitutional Committee votes, plus SPO votes for hard forks (similarly to the previously defined, and now obsoleted, `CIP-47`).

The remainder of this proposal will be implemented in the second stage.


## Copyright

This CIP is licensed under [CC-BY-4.0](https://creativecommons.org/licenses/by/4.0/legalcode)

[^1]: A formal description of the current rules for governance actions is given in the [Shelley ledger specification](https://hydra.iohk.io/job/Cardano/cardano-ledger/shelleyLedgerSpec/latest/download-by-type/doc-pdf/ledger-spec).

    - For protocol parameter changes (including hard forks), the `PPUP` transition rule (Figure 13) describes how protocol parameter updates are processed, and the `NEWPP` transition rule (Figure 43) describes how changes to protocol parameters are enacted.

    - For funds transfers, the `DELEG` transition rule (Figure 24) describes how MIR certificates are processed, and the `MIR` transition rule (Figure 55) describes how treasury and reserve movements are enacted.

    > **Note**
    > The capabilities of the `MIR` transition rule were expanded in the [Alonzo ledger specification](https://hydra.iohk.io/job/Cardano/cardano-ledger/specs.alonzo-ledger/latest/download-by-type/doc-pdf/alonzo-changes)


[^2]: There are many varying definitions of the term "hard fork" in the blockchain industry. Hard forks typically refer to non-backwards compatible updates of a network. In Cardano, we formalize the definition slightly more by calling any upgrade that would lead to _more blocks_ being validated a "hard fork" and force nodes to comply with the new protocol version, effectively obsoleting nodes that are unable to handle the upgrade.

[^3]: This is the definition used in "active stake" for stake delegation to stake pools, see Section 17.1, Total stake calculation, of the [Shelley ledger specification](https://hydra.iohk.io/job/Cardano/cardano-ledger/shelleyLedgerSpec/latest/download-by-type/doc-pdf/ledger-spec).
