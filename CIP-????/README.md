---
CIP: ?
Title: Cardano Ledger Evolution
Status: Proposed
Category: Meta
Authors: Jared Corduan <jared.corduan@iohk.io>
Implementors: N/A
Created: 2023-01-30
License: CC-BY-4.0
---

# Evolution of the Cardano ledger

## Abstract

This CIP provides guidance for future CIPs concerning the Cardano ledger.

## Motivation

The ledger is responsible for processing transactions and updating the shared state of the network.
It also processes block headers and handles the state transformation from one epoch to the next
(e.g. computing the staking rewards).

Most of the state maintained by the ledger relates to the
[Extended UTxO accounting model](https://iohk.io/en/research/library/papers/the-extended-utxo-model/) and
support for [Ouroboros](https://iohk.io/en/research/library/papers/ouroboros-a-provably-secure-proof-of-stake-blockchain-protocol/),
the proof-of-stake consensus mechanism used in Cardano.

This CIP aims to give guidance for future CIPs related to the ledger,
making it a registered category of the CIP process[^1].
[^1]: See [CIP-1](https://github.com/cardano-foundation/CIPs/blob/cip-cps-rework/CIP-0001/README.md#categories).
While nothing new is added to the usual CIP process (except for a ledger registry),
expectations for ledger CIPs are made explicit and some background information is provided.

Many thanks to Arnaud Bailly and Michael Peyton Jones for all their help reviewing and providing
feedback on the first versions of this CIP.

## Background

### Terminology

Context for the terminology used in this document is given in [CIP-59].

### Specifications

The ledger is specified as a state transition system using a
[small step operational semantics](https://github.com/input-output-hk/cardano-ledger/releases/latest/download/small-step-semantics.pdf).
We refer to this framework as the *Small Step Semantics for Cardano*, or the *STS* for short.
An understanding of the existing STS specifications for the
existing ledger eras is often required to fully understand the implications of changes to the ledger
(though an understanding of the Haskell implementation is a fair substitute).

The STS framework leaves both cryptographic primitives and the serialization format abstract.
The STS specifications need to be complete enough to realize a full implementation of the ledger
given the cryptographic primitives and serialization format.
The cryptographic primitives are described as appendices to the STS specifications,
and the serialization format is given as a
[CDDL file](https://www.rfc-editor.org/rfc/rfc8610).
There SHOULD be one STS specification per ledger era.

From the Byron to the Babbage ledger eras, the STS frameworks were written in $\LaTeX$.
Starting in the Conway ledger era, literate Agda will be used.
During the transition from $\LaTeX$ to literate Agda, we will take advantage
of the ability to substitute $\LaTeX$ in place of Agda code when needed for expedience.
With time, the Agda specification will not only be used to provide PDF specifications,
but also reference implementations.

### Ledger eras

A ledger era is a collection of features added to the ledger which are introduced at a hard fork.
The existing ledger eras, with very simplistic descriptions, are given below.

|name|new features|link|
| --- | --- | --- |
|Byron|initial UTxO ledger|[spec](https://github.com/input-output-hk/cardano-ledger/releases/latest/download/byron-ledger.pdf)|
|Shelley|decentralized block production, stake delegation|[spec](https://github.com/input-output-hk/cardano-ledger/releases/latest/download/shelley-ledger.pdf)|
|Allegra|timelock scripts|-|
|Mary|multi-assets|[spec](https://github.com/input-output-hk/cardano-ledger/releases/latest/download/mary-ledger.pdf)|
|Alonzo|Plutus scripts|[spec](https://github.com/input-output-hk/cardano-ledger/releases/latest/download/alonzo-ledger.pdf)|
|Babbage|improved Plutus script contexts|[spec](https://github.com/input-output-hk/cardano-ledger/releases/latest/download/babbage-ledger.pdf)|
|Conway|governance|[spec WIP](https://github.com/input-output-hk/formal-ledger-specifications)|

Note that there is no Allegra specification.
The Allegra era consists entirely of the addition of timelocks to the MultiSig script
introduced in the Shelley ledger era
(See figure 12 of the Mary specification).

Note that small, isolated changes can be made within a ledger era
by way of an intra-era hard fork. See [CIP-59] for more details.

#### Ledger events

Some provenance about the ledger calculations is provided by ledger events.
Sometimes these events have clear triggers (e.g. Plutus script execution)
and sometimes they provide intermediate calculations performed by the ledger rules
(e.g. the reweard calculation).
The events come with zero cost to a running node if not used and are not stored in the ledger state.
Documentation about the existing events can be found
[here](https://github.com/input-output-hk/cardano-ledger/blob/master/docs/LedgerEvents.md).

### Soft forks and Hard forks

Since most ledger CIPs will involve backwards incompatible changes,
the following two definitions are helpful:

**Hard fork** - A *hard fork* is a change to the protocol (not restricted to the ledger)
resulting in a single new block definition becoming valid.

Alternatively, a hard fork is a backwards incompatible change for both
block producers and block validators.

**Soft fork** - A *soft fork* is a change to the protocol (not restricted to the ledger)
resulting in fewer blocks being valid.

Alternatively, a soft fork is a backwards incompatible change for
block producers, but a backwards compatible change for block validators.

### Serialization

Transactions and blocks are serialized with
[CBOR](https://www.rfc-editor.org/rfc/rfc7049)
and specified with
[CDDL file](https://www.rfc-editor.org/rfc/rfc8610).

Serialization changes to the ledger are discussed in
[CIP-80](https://github.com/cardano-foundation/CIPs/pull/372).

Note that the serialisation format of the ledger state is unspecified and left as an
implementation detail (unlike the format of blocks).

### The ledger-script interface

The ledger and Plutus scripts have a common interface, described in [CIP-35].
CIPs relating to this inteface are relevant to both the ledger and to the Plutus CIP categories.

## Specification

### What merits a ledger CIP?

The criterion for deciding if a change to the ledger merits a CIP is as follows:
changes to the ledger require going through the CIP process whenever
every implementation of the Cardano ledger needs to be standardized on the details.

Bug fixes are an exception to this criterion, they do not merit a CIP except in the case
that the fix is substantially complicated.
A "bug fix" is a change to behavior where:
- The implemented behavior does not match the specification; or
- The specified behavior is clearly wrong (in the judgment of relevant experts)

Serialization changes are another possible exception to the criterion.
Many serialization changes can be handled as a part of the normal development process
without the need for a CIP.
Dramatic changes to the serialization, however, may benefit from the CIP process.

The ledger rules MUST be standardized in order for consensus to be maintained,
but things like the ledger events are more open to debate.

Changes to the protocol parameter values do not require a CIP since they are
a governance issue (see [CIP-1694]).

### Ledger Core CIP registry

Any CIP which proposes a change to the ledger MUST be added to this table:

| #    | Title                                       | Status   |
|---   | ---                                         | ---      |
|   31 | Reference inputs                            | Active   |
|   32 | Inline datums                               | Active   |
|   33 | Reference scripts                           | Active   |
|   59 | Terminology Surrounding Core Features       | Active   |
|   80 | Transaction Serialization Deprecation Cycle | Proposed |
| 1694 | A proposal for entering the Voltaire phase  | Proposed |

### Expectations for ledger CIPs

* Familiarity with the existing ledger specifications is required to propose changes to the ledger.
* The CIP specifications for ledger CIPs must be sufficiently detailed for inclusion in
  a formal ledger specification.
* Some changes may require getting an expert opinion.
  For example, changes to the incentives model (such as changing the reward calculation)
  should include a proper game theoretic analysis.

## Rationale

### There is only one implementation, why limit the scope of ledger CIPs in this way?

Even though there is currently only one implementation, this provides us with a clear
definition of what is essential to the ledger.
It also provides a clear path for future implementations.

### Why is the specification vague about the role of ledger events in the CIP process?

This decision should be left to the community as more use cases emerge.

### Why is familiarity with the formal specifications required?

It is not always clear which seemingly small details can make a large difference
to the many consumers of the ledger.
It is better that the CIP process achieve consensus on all the details than for
these decisions to be made during the implementation phase.


[CIP-35]: https://github.com/cardano-foundation/CIPs/tree/master/CIP-0035
[CIP-59]: https://github.com/cardano-foundation/CIPs/tree/master/CIP-0059
[CIP-1694]: https://github.com/cardano-foundation/CIPs/tree/master/CIP-1694