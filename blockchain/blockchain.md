*** This is the beginning of an unfinished draft. Don't continue reading! ***

# Tendermint Blockchain

> Rough outline of what the component is doing and why. 2-3 paragraphs

A blockchain is a growing list of sets of transactions, denoted by
*chain*.  If several processes in a distributed system have access to
the blockchain, they can (1) provide transactions as input and (2) use
*chain* to execute these transactions in order.

The *chain* itself should be implemented in a reliable way, which
introduces the need for fault-tolerance and distribution.  The
Tendermint protocols implement the blockchain over an unreliable
(Byzantine) distributed system.  More precisely, a Tendermint system
consists of many so-called [full nodes][fullnode] that each maintain a
local copy of (a prefix of) the current *chain*.

In this specification, we are only concerned with the *chain*.
They are maintained at the (correct) full nodes, and are the result of the
execution of the Tendermint consensus protocol, described in the
[ArXiv paper][arXiv]. The  *chain* is called *decision* in the
paper, and the paper does not consider internals of what is stored in
decision. The Tendermint consensus implements a loop with iterator
*h* (height).  In each iteration, upon deciding on the transactions to
be put into the *chain*, a new *chain* entry is created.


# Part I - Outside view

## Context of this document

> mention other components and or specifications that are relevant for this
spec. Possible interactions, possible use cases, etc.

> should give the reader the understanding in what environment this component
will be used.

This specification is central in the collection of Tendermint
protocols.  The behavior of protocols like [fastsync][fastsync], or
the [light client][lightclient] will be defined with respect to this
specification.  E.g., the light client implements a read operation of
the *chain* entry of some height *h*.  It is thus crucial to
understand what data is stored in this *chain* entry, and what
are the precise semantics of a read operation in a faulty environment.


## Informal Problem statement

> for the general audience, that is, engineers who want to get an overview over what the component is doing
from a bird's eye view.

A blockchain provides a growing sequence of sets of transactions.

## Sequential Problem statement

> should be English and precise. will be accompanied with a TLA spec.

#### **[TMBC-HEADER]**:
A set of blockchain transactions is stored in a data structure called
*block*, which contains a field called *header*. (The data structure
*block* is defined [here][block]).  As the header contains hashes to
the relevant fields of the block, for the purpose of this
specification, we will assume that the blockchain is a list of
headers, rather than a list of blocks. 

#### **[TMBC-HASH-UNIQUENESS]**:
We assume that every hash in the header identifies the data it hashes. 
Therefore, in this specification, we do not distinguish between hashes and the 
data they represent.


#### **[TMBC-HEADER-Fields]**:
A header contains the following fields:

 - `Height`: non-negative integer
 - `Time`: time (integer)
 - `LastBlockID`: Hashvalue
 - `LastCommit` DomainCommit
 - `Validators`: DomainVal
 - `NextValidators`: DomainVal
 - `Data`: DomainTX
 - `AppState`: DomainApp
 - `LastResults`: DomainRes


#### **[TMBC-SEQ]**:

The Tendermint blockchain is a list *chain* of headers. 

### Appending a block

#### **[TMBC-SEQ-GROW]**: 

During operation, new headers may be appended to the list one by one.



In the following, *ETIME* and *LTIME* are a lower and upper bounds,
respectively, on the time interval between the times at which two
successor blocks are added. We are not fixing these times
here. Rather, they should serve as defining constraints for temporal
properties of other protocols. For instance, we might instantiate
these times by setting *ETIME* to infinity, when we say that
*Fastsync* terminates in the case the blockchain does not grow.

#### **[TMBC-SEQ-APPEND-E]**: 
If a header is appended at time *t* then no additional header will be
appended before time *t + ETIME*.


#### **[TMBC-SEQ-APPEND-L]**: 
If a header is appended at time *t* then the next header will be
appended before time *t + LTIME*.

#### **[TMBC-SEQ-APPEND-ELEL]**: 
*ETIME <= LTIME*

 



 
### Basic Soundness Conditions
 


#### **[TMBC-SOUND-INC-HEIGHT]**:
 For all *i < len(chain)*: *chain[i].Height + 1 = chain[i+1].Height*

*Remark:* We do not write *chain[i].Height = i*, to allow that a chain
can be started at some arbitrary height, e.g., when there is social
consensus to restart a chain from a given height/block.



#### **[TMBC-SOUND-INC-TIME]**:
For all *i < len(chain)*: *chain[i].Time < chain[i+1].Time*


#### **[TMBC-SOUND-NextV]**:
For all *i < len(chain)*: *chain[i+1].Validators = chain[i].NextValidators*




###  Functions, Domains, and more soundness conditions

#### **[TMBC-SOUND-PossCommit]**:
There is a function *PossibleCommit* that maps a block (header) to a set
of values  in DomainCommit, cf. [TMBC-HEADER-Fields].

<!---
There is a function PossibleCommit that maps for *0 < i <= len(chain)*, 
chain[i-1] to a set of values in DomainCommit.
-->


#### **[TMBC-SOUNDNESS-FUNCTIONS]**:
The system provides the following functions:

- `hash`: We assume that every hash identifies the data it hashes

- `execute`: used for state machine replication. maps *Data*
  (transactions) and an *application state* to a new state. It is a function
  (deterministic transitions).  
  **TODO:** it is provided by the
  application. Do we need to talk about applications in this spec?

- `proof(b,commit)`: a predicate: true iff 
     * *b* is part of the *chain*, that is, there is an *i* such that
       *chain[i] = b*
	 * *commit* is in PossibleCommit(b), cf. [TMBC-SOUND-PossCommit].

*Remark.* Observe that *proof* refers to the *chain*. It thus depend
on the execution, which results in a different quantifier order. For
instance, we say "there exists a function *hash* such that for all
runs", while we say "for each run there exists a function
*proof*". The consequence is that *hash* is a predetermined function
(implemented), while *proof* will have to be computed during the run
as a function of the *chain*. The challenge in a distributed system is
to locally compute *proof* without necessarily having complete
knowledge of *chain*. In the context of the light client, we even want
to infer knowledge about *chain* from the outcomes of the local
computation of *proof*. We will use digital signatures for that. We
will introduce them below when we introduce the distributed aspects.


#### **[TMBC-SOUNDNESS-PREDICATES]**:
Given two blocks *b* and *b'*:

- `match-hash(b,b')` iff *hash(b) = b'.LastBlockID*
- `match-proof(b,b')` iff *proof(b, b'.LastCommit)*

#### **[TMBC-SOUND-CHAIN]**:
 For all *i < len(chain)*: *match-hash(chain[i], chain[i+1])*
 
#### **[TMBC-SOUND-LAST-COMM]**:
For all *i < len(chain)*: *match-proof(chain[i], chain[i+1])*


#### **[TMBC-SOUND-APP]**:
For all *i < len(chain)*: *chain[i+1].AppState = execute(chain[i].Data,chain[i].AppState)*


#### **[TMBC-SEQ-INV]**

At all times, the chain is sound [TMBC-SOUND-*].




 



 
# Part II - Protocol view

## Environment/Assumptions/Incentives

> Introduce distributed aspects

> Timing and correctness assumptions. Possibly with justification that the
assumptions make sense, e.g., it is in the interest of a full node to behave
correctly

> should have clear formalization in temporal logic.

## Timing, nodes, and correctness assumptions

In a Tendermint system, the nodes that choose to interact with each
other collectively compute the  *chain* in a distributed manner
by executing the [consensus protocol][arXiv]. 
The Tendermint protocols should ensure that
the participating nodes cannot benefit by deviating from the expected
behavior.  As a result, the nodes should be motivated (incentivized)
to follow the protocol.  This is achieved by requiring the nodes to
bond atoms and by the system punishing the nodes that do not follow
the protocol.  That is, the nodes bond atoms in order to participate,
and they are punished if they misbehave.  If the nodes want to claim
their atoms, they have to wait for a certain period of time, called
the *unbonding period*.  The unbonding period is a configuration
parameter of the system.

#### **[TMBC-TIME-PARAMS]**:
A Tendermint blockchain has the following configuration parameters:
 - *unbondingPeriod*: a time duration.
 - *trustingPeriod*: a time duration smaller than *unbondingPeriod*.




#### **[TMBC-NODES]**:
Tendermint full nodes (or just *full nodes*), execute a set of
protocols, e.g., consensus, gossip, fast sync, etc.  When a full node
actively participates in the distributed consensus, it is called a
*validator node*.

#### **[TMBC-CORRECT]**:
We define a predicate *correctUntil(n, t)*, where *n* is a node and *t* is a 
time point. 
The predicate *correctUntil(n, t)* is true if and only if the node *n* 
follows all the protocols (at least) until time *t*.

## Authenticated Byzantine Model

Due to the incentives, we claim that it is safe to assume below that
most of the full nodes will follow the protocol. However, we also have
to capture that full nodes deviate from the prescribed behavior,
either due to misconfiguration or implementation bugs, or because of
adversarial behavior. At the same time, we will heavily use digital
signature, and they constitute a stable technology that allows to
determine the sender of a message, even if a messages is wrapped into
another message and forwarded. In the distributed algorithm literature
(e.g., [[DLS88]][DLS]), this model is called **authenticated
Byzantine**.  Similar to Nancy Lynch when writing her book on
distributed algorithms, "we do not know of a nice formal definition"
for Byzantine failures with authentication.  So, we give an
axiomatization that is sufficient for verification.

#### **[TMBC-Sign]**:
- For each node with address *addr*, there is a function *sign_addr*
  that maps *Data* to a domain *SignedData(addr)*.
- *SignedDataDomain* is the disjoint union of *SignedData(addr)* over
  all *addr*. We call an element of *SignedDataDomain* signed data.
- There is a function *check* that maps signed data *sd*
  to the pair *(data, addr)* such that *sd = sign_addr(data)*.

#### **[TMBC-Sign-NoForge]**:
For all runs *r*, for all nodes *p* and *q*, for all
*sdq* from SignedDataDomain, if *aq* is the address of *q* and *p*
(different from *q*)
sends a message that contains *sdq* from *SignedData(aq)* in run *r*,
then *q* has sent a message containing *sdq* earlier in run *r*.

*Remark:* [TMBC-Sign-NoForge] can be written as invariant over the
message history.


#### **[TMBC-FaultyFull]**:
No assumption is made about the internal
behavior of faulty full nodes.

#### **[TMBC-Auth-Byz]**:
The authenticated Byzantine model assumes [TMBC-Sign-NoForge] and
[TMBC-FaultyFull], that is, faulty nodes are limited in that they
cannot forge messages [TMBC-Sign-NoForge].




## Validators

In [TMBC-HEADER-Fields], most of the fields are defined for abstract
domains. Here we will specialize DomainVal and DomainCommit to the
distributed setting, and describe how validators and commit are
implemented in Tendermint consensus.

*Remark:* We observed that in the existing documentation the term
*validator* refers to both a data structure and a full node that
participates in the distributed computation. Therefore, we introduce
the notions *validator pair* and *validator node*, respectively, to
distinguish these notions in the cases where they are not clear from
the context.


#### **[TMBC-VALIDATOR-Pair]**:

Given a full node, a 
*validator pair* is a pair *(address, voting_power)*, where 
  - *address* is the address (public key) of a full node, 
  - *voting_power* is an integer (representing the full node's
  voting power in a certain consensus instance).
  
*Remark:* In the Golang implementation the data type for *validator
pair* is called `Validator`


#### **[TMBC-VALIDATOR-Set]**:

A *validator set* is a set of validator pairs. For a validator set
*vs*, we write *TotalVotingPower(vs)* for the sum of the voting powers
of its validator pairs.

#### **[TMBC-VP-SOUND-VALID-UNIQUE]**:
For each block in the chain, the set *Validators* contains at most 
one validator pair for each full node.

#### **[TMBC-VP-SOUND-NEXT-VALID-UNIQUE]**:
For each block in the chain, the set *NextValidators* contains at 
most one validator pair for each full node.
 


## Distributed Definition of Commit



#### **[TMBC-VOTE]**:
A *vote* contains a `prevote` or `precommit` message sent and signed by
a validator node during the execution of [consensus][arXiv]. Each 
message contain the following fields
   - `Type`: prevote or precommit
   - `Height`: positive integer
   - `Round` a positive integer
   - `BlockID` a Hashvalue of a block (not necessarily a block of the chain)


#### **[TMBC-COMMIT]**:
A commit is a set of votes.

**TODO:** clarify whether `prevote` or `precommit` are equivalent in
the Commit.

#### **[TMBC-SOUND-DISTR-PossCommit]**:
For a block *b*, each element *pc* of *PossibleCommit(b)* satisfies:
  - each vote *v* in *pc* satisfies
     * *pc* contains only votes (cf. [TMBC-VOTE])
	 by validators from *b.Validators*
     * v.blockID = hash(b)
	 * v.Height = b.Height  
	 **TODO:** complete the checks here
  - the sum of the voting powers in *pc* is greater than 2/3
  *TotalVotingPower(b.Validators)*


#### **[TMBC-SOUND-DISTR-LAST-COMM]**:
Combining the specialization of *PossibleCommit* from
[TMBC-SOUND-DISTR-PossCommit] with the abstract definitions
[TMBC-SOUND-FUNCTIONS] and [TMBC-SOUND-LAST-COMM] we obtain the
definition of soundness for LastCommit in the distributed setting.
	 





## Commit Invariants

Commit messages are used to establish proof that a certain block is on
the blockchain. 

We now make explicit some invariants a correct validator node must ensure.
We us from the [consensus algorithm][arXiv]
the predicate `valid` over blocks, and `precommit`
     messages.
Correct validator nodes use *valid* to ensure the soundness requirements of
     the blockchain [TMBC-SOUND-?], and send *precommit* messages
     only for blocks for which *valid* evaluates to true.

#### **[TMBC-INV-CORR-PROC-VALID]**:

There is a predicate `valid` over a block (and the prefix of the
chain, which we omit in the notation).
In particular, if *valid(b)* evaluates to true at a correct validator node,
     then *b.Validators = pred'.NextValidators* of the block *pred* of
     height *h - 1* of the blockchain;


The following invariant is crucial to guarantee the soundness of the chain:


#### **[TMBC-INV-CORR-PROC-COMMIT]**:

A correct validator node sends and signs precommit for a block *b*, only if `valid(b)`.

*Remark:* Follows from code line 36 in the  [consensus algorithm][arXiv].





From [TMBC-INV-CORR-PROC-VALID] and [TMBC-INV-CORR-PROC-COMMIT]
follows that **more than two thirds of the voting
power in *b.Validators* is correct for any block *b* signed by a correct
validator node**. As a result, a commit that is well-formed (that is, is in
*PossibleCommit(b)*) and signed by a correct validator node is a proof that
*b* is in the blockchain. 

*Remark:* "Signed by a correct validator node" means that the
validator node *n*
sends *precommit* at time *t* and *correctUntil(n, t)* holds.



#### **[TMBC-VAL-COMMIT]**:

If for a block *b*,  a commit *c*
  - contains at least one validator pair *(v,p)* such that *v* is a correct
    validator node, and
  - is contained in *PossibleCommit(b)*
  
then the block *b* is on the blockchain.




## Tendermint failure model (a.k.a. Tendermint Security Model)





#### **[TMBC-FM-2THIRDS]**:
If a block *h* is in the chain,
then there exists a subset *CorrV*
of *h.NextValidators*, such that:
  - *TotalVotingPower(CorrV) > 2/3
    TotalVotingPower(h.NextValidators)*; cf. [TMBC-VALIDATOR-Set]
  - For every validator pair *(n,p)* in *CorrV*, it holds *correctUntil(n,
    h.Time + trustingPeriod)*; cf. [TMBC-CORRECT]



<!--- Formally,
\[
\sum*{(v,p) \in h.Header.NextV \wedge correctUntil(v,h.Header.bfttime + trustingPeriod)} p >
2/3 \sum*{(v,p) \in h.Header.NextV} p
\] -->

*Remark:* The definition of correct
[**[TMBC-CORRECT]**](TMBC-CORRECT-link) refers to realtime, while it
is used here with *Time* and *trustingPeriod*, which are "hardware
times".  We do not make a distinction here.


From [TMBC-FM-2THIRDS] we directly derive the following observation:

#### **[TMBC-VAL-CONTAINS-CORR]**:

Given a (trusted) block *tb* of the blockchain, a set of full nodes 
*N* contains a correct node at a real-time *t*, if
   - *t - trustingPeriod < tb.Time < t*
   - the voting power in tb.NextValidators of nodes in *N* is more
     than 1/3 of *TotalVotingPower(tb.NextValidators)*





*Remark:* The light client verification checks [TMBC-VAL-CONTAINS-CORR] and
[TMBC-VAL-COMMIT] as follows: 
Given a trusted block *tb* and an untrusted block *ub* with a commit *cub*,
one has to check that *cub* is in *PossibleCommit(ub)*, and that *cub*
contains a correct node using *tb*.



Until now, we have established soundness of the blockchain, and some
invariants expected from correct validators when observed from the
outside. Below we describe the internals of the [consensus
algorithm][arXiv]. For details we refer to the [paper][arXiv].  
*Remark:* For
now the goal of this specification is to have a formal understanding of the outside view of
the blockchain in order to be able to specify other protocols.

## Distributed Problem Statement

### Design choices


#### **[TMBC-FM-CONS]**: 
(Consensus failure model)
There is a set *C* of validator pairs, such that *C* is a subset of *NextValidators* at height *h*, where: 
  - The validator pairs in *C* hold more than two-thirds of the total voting power in *NextValidators* at height *h*
  - For every validator pair *(n,p)* in *C*, follows the consensus protocol until consensus for height *h+1* is terminated. 


We recall that [TMBC-CORRECT] denotes by *correctUntil(n, t)* that full
node *n* is correct up to time *t* if it follows all the protocols
up to time *t*. For now we assume that both failure assumptions
[TMBC-FM-2THIRDS] and [TMBC-FM-CONS] hold.



#### **[TMBC-LOCAL-CHAIN]**:

Each correct full node *p* maintains its local copy of a prefix the Tendermint
blockchain, denoted by *chain_p*.



### Temporal Properties

> safety specifications / invariants in English

> liveness specifications in English. Possibly with timing/fairness requirements:
e.g., if the component is connected to a correct full node and communication is
reliable and timely, then something good happens eventually.





### Safety

#### **[TMBC-VC-AGR]**:
At all times *t*, for any two full nodes *p* and *q*, with *correctUntil(p,
     t)* and *correctUntil(q, t)* it holds that *chain_p(t)* is a prefix
     of *chain_q(t)* or *chain_q(t)* is a prefix of *chain_p(t)*.

#### **[TMBC-VC-VAL]**:
For a full node *p*, we substitute *chain* with *chain_p* in the
soundness properties [TMBC-SOUND-?]. For all times *t* and every full
node *p*, with *correctUntil(p, t)*, the soundness requirements hold for
*chain_p(t)*.


*Remark:* Validity [TMBC-VC_VAL] should make reference to the mempool,
e.g., only messages from the mempool are added to Data + we will need
a spec for the mempool. For now I leave it like that as the light
client and fastsync do not care about that.
  
*Remark:* Additional application specific soundness requirements might
also need to hold.



### Liveness

The following is an abstract liveness property that states that a
correct full nodes infinitely often append new blocks to the
chain. This can be defined only for full nodes that are correct
forever (as one needs infinite traces). 


#### **[TMBC-VC-LIVE]**: 
For all full nodes *p*, with *correctUntil(p, infinity)*, for all times
*t*, there exists a time *t'*, such that *|chain_p(t)| <
|chain_p(t')|*.


The following property is formally not a liveness property (as it can
be violated on a finite prefix) but is a progress property of practical
relevance: 

#### **[TMBC-VC-PROG]**:
For all full nodes *p*, and all times *t'*: If *correctUntil(p, t')*,
then for all times *t < t' - LTIME* it holds that *|chain_p(t)| <
|chain_p(t + LTIME)|*.

*Remark:* In the temporal properties above we use the *correctUntil*
predicate, in a way that suggests that all full nodes participate in
Tendermint since the genesis block. However, there are Tendermint
protocols (state sync, fast sync) that allow nodes to join the system
later. We will have to define later what it means for these nodes to satisfy
[TMBC-VC_AGR] and [TMBC-VC_VAL] and [TMBC-VC-PROG]. For instance, they
may not need to have the complete prefix of *chain* but start at some height.


> How is the problem statement linked to the "Sequential Problem statement".
Simulation, implementation, etc. relations

### Solving the sequential specification

**TODO:** How does the distributed specification map to the sequential
one? For instance, at each time the longest prefix of *chain_p* for
some *p* defines *chain* in the sequential specification.

#### **[TMBC-CorrFull]**: 
Every correct Tendermint full node locally stores a prefix of the
current list of headers from [**[TMBC-SEQ]**](TMBC-SEQ-link).




**For the remainder, we refer to the [arXiv paper](arXiv) for now.**


## Definitions

> In this section we become more concrete, with basic (abstracted) data types

> some math that allows to write specifications and pseudo code solution below.
Some variables, etc.

### Data structures



## Solution

> Basic data structures. Simplified, so that we can focus on the distributed
algorithm here. If existing: link to Tendermint data structures, and mentioned
if details were omitted.



### Outline

> Describe solution (in English), decomposition into functions, where communication to other components happens.

### Details

> Pseudo code of the solution


## Correctness arguments

> Proof sketches of why we believe the solution satisfies the problem statement.
Possibly giving inductive invariants that can be used to prove the specifications
of the problem statement



# References

[[block]] Definition of the block data structure

[[blockchain]] Tendermint Blockcahin specification

[[fastsync]] Specification of the fastsync protocol

[[fullnode]] Specification of the full node API

[[header]] Definition of the header data structure

[[lightclient]] Light Client ADR

[[verifier]] Light Client Verification Specification

[[arXiv]] The Tendermint paper on arXiv


[block]: https://github.com/tendermint/spec/blob/master/spec/blockchain/blockchain.md#block 
[blockchain]: https://github.com/tendermint/spec/blob/master/spec/blockchain/blockchain.md#blockchain
[fastsync]: https://github.com/informalsystems/VDD/blob/master/fastsync/fastsync.md
[lightclient]: https://github.com/interchainio/tendermint-rs/blob/e2cb9aca0b95430fca2eac154edddc9588038982/docs/architecture/adr-002-lite-client.md#adr-002-light-client
[verifier]: https://github.com/informalsystems/VDD/blob/master/lightclient/verification.md#core-verification
[header]: https://github.com/tendermint/spec/blob/master/spec/blockchain/blockchain.md#header
[fullnode]: https://github.com/tendermint/spec/blob/master/spec/blockchain/fullnode.md

[TMBC-SEQ-link]: https://github.com/informalsystems/VDD/blob/master/lightclient/blockchain.md#tmbc-seq
[TMBC-VALIDATOR-link]: https://github.com/informalsystems/VDD/blob/master/lightclient/blockchain.md#tmbc-validator
[TMBC-CORRECT-link]: https://github.com/informalsystems/VDD/blob/master/lightclient/blockchain.md#tmbc-correct
[TMBC-TIME-link]: https://github.com/informalsystems/VDD/blob/master/lightclient/blockchain.md#tmbc-time

[arXiv]: https://arxiv.org/abs/1807.04938

[DLS]: https://groups.csail.mit.edu/tds/papers/Lynch/jacm88.pdf
