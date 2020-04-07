# Fastsync

<!--
> Rough outline of what the component is doing and why. 2-3 paragraphs 
---->

This document is the English specification of the *Fastsync*
protocol. It consists of the following parts:

- [Part I](#part-i---outside-view): Informal introduction; high-level (sequential) specification of the problem
  addressed by *Fastsync*
  
- [Part II](#part-ii---protocol-view): Protocol view, 
    - temporal logic specifications, 
	- description of Fastsync V2 (the protocol underlying the current Golang
      implementation)
	- analysis of Fastsync V2 that highlights several issues that
      prevent achieving some of the desired fault-tolerance properties

- [Part III](#part-iii---suggestions-for-an-improved-fastsync-implementation): some suggestions on how to address the issues in the future
  

# Part I - Outside view

## Context of this document

<!--
> mention other components and or specifications that are relevant for this
spec. Possible interactions, possible use cases, etc. 
---->
<!--
> should give the reader the understanding in what environment this component
will be used. 
---->

Fastsync is a protocol that is used by a full node to catch-up to the
current state of a Tendermint blockchain. Its typical use case is a
full node that was disconnected from the system for some time. The
recovering full node locally has a copy of a prefix of the blockchain,
and the corresponding application state that is slightly outdated. It
then queries its peers for the blocks that were decided on by the
Tendermint blockchain during the period the full node was
disconnected. After receiving these blocks, it executes the
transactions in the blocks in order to catch-up to the current height
of the blockchain and the corresponding application state.

In practice it is sufficient to catch-up only close to the current
height: The Tendermint consensus reactor implements a similar
functionality and can synchronize a node that is approximately 100
blocks away from the current height of the blockchain. Fastsync should
bring a node within this range.

### Blockchain

> We will briefly list some of the notions
> of blockchains as required for this specification. More details can
> be found  [here][block].

#### **[TMBC-HEADER]**:
A set of blockchain transactions is stored in a data structure called
*block*, which contains a field called *header*. (The data structure
*block* is defined [here][block]).  As the header contains hashes to
the relevant fields of the block, for the purpose of this
specification, we will assume that the blockchain is a list of
headers, rather than a list of blocks. 

#### **[TMBC-SEQ]**:

The Tendermint blockchain is a list *chain* of headers. 

#### **[TMBC-SEQ-GROW]**: 

During operation, new headers may be appended to the list one by one.


> In the following, *ETIME* is a lower bound
> on the time interval between the times at which two
> successor blocks are added. 

#### **[TMBC-SEQ-APPEND-E]**: 
If a header is appended at time *t* then no additional header will be
appended before time *t + ETIME*.


#### **[TMBC-Auth-Byz]**:
The authenticated Byzantine model assumes that no node (faulty or
correct) may break digital signatures and in addition to this, no
assumption is made about the internal behavior of faulty full
nodes. That is, faulty nodes are limited in that they cannot forge
messages.

<!-- The authenticated Byzantine model assumes [TMBC-Sign-NoForge] and -->
<!-- [TMBC-FaultyFull], that is, faulty nodes are limited in that they -->
<!-- cannot forge messages [TMBC-Sign-NoForge]. -->

#### **[TMBC-VALIDATOR-Set]**:

A *validator set* is a set of validator pairs. For a validator set
*vs*, we write *TotalVotingPower(vs)* for the sum of the voting powers
of its validator pairs.

#### **[TMBC-CORRECT]**:
We define a predicate *correctUntil(n, t)*, where *n* is a node and *t* is a 
time point. 
The predicate *correctUntil(n, t)* is true if and only if the node *n* 
follows all the protocols (at least) until time *t*.

#### **[TMBC-TIME-PARAMS]**:
A blockchain has the following configuration parameters:
 - *unbondingPeriod*: a time duration.
 - *trustingPeriod*: a time duration smaller than *unbondingPeriod*.

#### **[TMBC-FM-2THIRDS]**:
If a block *h* is in the chain,
then there exists a subset *CorrV*
of *h.NextValidators*, such that:
  - *TotalVotingPower(CorrV) > 2/3
    TotalVotingPower(h.NextValidators)*; 
  - For every validator pair *(n,p)* in *CorrV*, it holds *correctUntil(n,
    h.Time + trustingPeriod)*.

#### **[TMBC-CorrFull]**: 
Every correct full node locally stores a prefix of the
current list of headers from [**[TMBC-SEQ]**](TMBC-SEQ-link).





## Informal Problem statement

<!--
> for the general audience, that is, engineers who want to get an overview over what the component is doing
from a bird's eye view. 
---->
A full node has as input a block of the blockchain at height *h* and
the corresponding application state (or the prefix of the current
blockchain until height *h*). It has access to a set *peerIDs* of full
nodes called *peers* that it knows of.  The full node uses the peers
to read blocks of the Tendermint blockchain (in a safe way, that is,
it checks the soundness conditions), until it has read the most recent
block and then terminates.


## Sequential Problem statement

<!--
> should be English and precise. will be accompanied with a TLA spec.
---->

*Fastsync* gets as input a block of height *h* and the corresponding
application state *s* that corresponds to the block and state of that
height of the blockchain [**[TMBC-SEQ]**][TMBC-SEQ-link], and produces
as output (i) a list *L* of blocks starting at height *h* to some height
*terminationHeight*, and (ii) the application state when applying the
transaction of the list *L* to *s*. Fastsync has to satisfy the following
properties [FS-Seq-?]:


#### **[FS-Seq-Live]**: 
*Fastsync* eventually terminates.

 
#### **[FS-Seq-Term]**:
Let *bh* be the height of the blockchain at the time *Fastsync* starts.
When *Fastsync* terminates, it outputs a list of all blocks from
height *h* to some height *terminationHeight >= bh - 1*,
[**[TMBC-SEQ]**][TMBC-SEQ-link].

> The above property is independent of how many blocks are added to the
> blockchain while Fastsync is running. It links the target height to the
> initial state. If Fastsync has to catch-up many blocks, it would be
> better to link the target height to a time close to the
> termination. This is capture by the following specification:

#### **[FS-Seq-Term-SYNC]**:
Let *eh* be the height of the blockchain at the time *Fastsync*
terminates.  There is a constant *D* such that when *Fastsync*
terminates, it outputs a list of all blocks from height *h* to some
height *terminationHeight >= eh - D*, [**[TMBC-SEQ]**][TMBC-SEQ-link].


#### **[FS-Seq-Inv]**:
Upon termination, the application state is the one that corresponds to
the blockchain at height *terminationHeight*.

> As the block of height *h + 1* is needed to verify the block of
> height *h* we highlight th following clarification of the
> termination height.


#### **[FS-Seq-Height]**: 
The returned value *terminationHeight* is the height of the block with the largest
height that could be verified. In order to do so, *Fastsync* needs the
Commit of the block at height  *terminationHeight + 1* in the blockchain.


# Part II - Protocol view

## Environment/Assumptions/Incentives

<!--
> Introduce distributed aspects 
---->
<!--
> Timing and correctness assumptions. Possibly with justification that the
assumptions make sense, e.g., it is in the interest of a full node to behave
correctly 
---->
<!--
> should have clear formalization in temporal logic.
---->


#### **[FS-A-NODE]**:
We consider a node *FS* that performs *Fastsync*.

#### **[FS-A-PEER-IDS]**:
*FS* has access to a set *peerIDs* of IDs (public keys) of peers (full
     nodes). During the execution of *Fastsync*, another protocol (outside
     of this specification) may add new IDs to *peerIDs*.



#### **[FS-A-PEER]**:
Peers can be faulty, and we do not make any assumptions about the number or
ratio of correct/faulty nodes. Faulty processes may be Byzantine
according to [**[TMBC-Auth-Byz]**][TMBC-Auth-Byz-link].

#### **[FS-A-VAL]**:
The system satisfies [**[TMBC-Auth-Byz]**][TMBC-Auth-Byz-link] and [**[TMBC-FM-2THIRDS]**][TMBC-FM-2THIRDS-link]. Thus, there is a
blockchain that satisfies the soundness requirements (that is, the
validation rules in [[block]]).

#### **[FS-A-COMM]**:
Communication between the node *FS* and all correct peers is reliable and
bounded in time: there is a message end-to-end delay *Delta* such that
if a message is sent at time *t* by a correct process to a correct
process, then it will be received and processed by time *t +
Delta*. This implies that we need a timeout of at least *2 Delta* for
remote procedure calls to ensure that the response of a correct peer
arrives before the timeout expires.

<!--
#### **[FS-A-LCC]**:
The node *FS* executing Fastsync is following the protocol (it is correct).
---->

## Distributed Problem Statement

### Design choices

<!--
> input/output variables used to define the temporal properties. Most likely they come from an ADR
---->

#### Two Kinds of Termination

We do not assume that there is a correct full node in
*peerIDs*. Under this assumption no protocol can guarantee the combination
of the properties [FS-Seq-Live] and
[FS-Seq-Term] and [FS-Seq-Term-SYNC] described in the sequential
specification above. Thus, in the (unreliable) distributed setting, we
consider two kinds of termination (successful and failure) and we will
specify below under what (favorable) conditions *Fastsync* ensures to
terminate successfully, and satisfy the requirements of the sequential
problem statement:

#### **[FS-DISTR-TERM]**:
*Fastsync* eventually terminates: it either *terminates successfully* or it  *terminates with failure*.


#### Remote Functions

The Tendermint Full Node exposes the following functions over
Tendermint RPC. The "Expected precondition" are only expected for
correct peers (as no assumption is made on internals of faulty
processes [FS-A-PEER]).


> In this document we describe the communication with peers 
via asynchronous RPCs.


```go
func Status(addr Address) (int64, error)
```
- Implementation remark
   - RPC to full node *addr*
- Expected precondition
  - none
- Expected postcondition
  - if *addr* is correct: Returns the current height `height` of the
    peer. [FS-A-COMM]
  - if *addr* is faulty: Returns an arbitrary height. [**[TMBC-Auth-Byz]**][TMBC-Auth-Byz-link]
- Error condition
   * if *addr* is correct: none. By [FS-A-COMM] we assume communication is reliable and timely.
   * if *addr* is faulty: arbitrary error (including timeout). [**[TMBC-Auth-Byz]**][TMBC-Auth-Byz-link]
----


 ```go
func Block(addr Address, height int64) (Block, error)
```
- Implementation remark
   - RPC to full node *addr*
- Expected precondition
  - 'height` is less than or equal to height of the peer
- Expected postcondition
  - if *addr* is correct: Returns the block of height `height`
  from the blockchain. [FS-A-COMM]
  - if *addr* is faulty: Returns arbitrary block [**[TMBC-Auth-Byz]**][TMBC-Auth-Byz-link]
- Error condition
  - if *addr* is correct: precondition violated. [FS-A-COMM]
  - if *addr* is faulty: arbitrary error (including timeout). [**[TMBC-Auth-Byz]**][TMBC-Auth-Byz-link]
----

### Temporal Properties


#### Fairness

As mentioned above, without assumptions on the correctness of some
peers, no protocol can achieve the required specifications. Therefore, 
we sometimes consider the following (fairness) constraint in the
safety and liveness properties below:


#### **[FS-SOME-CORR-PEER]**:
Initially, the set *peerIDs* contains at least one correct full node.

> While in principle the above condition can be part of a sufficient
> condition to solve [FS-Seq-Live] and
> [FS-Seq-Term] and [FS-Seq-Term-SYNC], we will see below that the
> current implementation of Fastsync (V2) requires the following (much
> stronger) requirement

#### **[FS-ALL-CORR-PEER]**:
At all times, the set *peerIDs* contains only correct full nodes.


#### Safety


<!--
> safety specifications / invariants in English 
---->


#### **[FS-VC-ALL-CORR-NONABORT]**:
Under [FS-ALL-CORR-PEER], *Fastsync* never terminates with failure.


#### **[FS-VC-STATE-INV]**:
If *FastSync* terminates successfully at height *terminationHeight*, then the
application state is the one that corresponds to the blockchain at
height *terminationHeight*.

#### **[FS-VC-BLOCKS-INV]**:
If *FastSync* terminates successfully at height *terminationHeight*, then the
returned list of blocks  is the one that corresponds to the blocks of
the
blockchain.


> As this specification does
> not assume that a correct peer is at the most recent height
> of the blockchain (it might lag behind), the property [FS-Seq-Term]
> cannot be ensured in an unreliable distributed setting. We consider
> the following relaxation. (Which is typically sufficient for
> Tendermint, as the consensus reactor then synchronizes from that
> height.)

#### **[FS-VC-CORR-INV]**:
Under [FS-ALL-CORR-PEER], let *maxh* be the maximum 
height of a correct peer [**[TMBC-CorrFull]**][TMBC-CorrFull-link]
in *peerIDs* at the time *Fastsync* starts. If *FastSync* terminates
successfully, it is at some height *terminationHeight >= maxh - 1*.

> The above property is independent of how many blocks are added to the
> blockchain (and learned by the peers) while *Fastsync* is running. It
> links the target height to the initial state. If *Fastsync* has to
> catch-up many blocks, it would be better to  link the
> target height to a time close to the termination. This is captured by
> the following specification:


#### **[FS-VC-CORR-INV-SYNC]**:
Under [FS-ALL-CORR-PEER], there exists a constant time interval *TD*, such
that if *term* is the time *Fastsync* terminates and
*maxh* be the maximum height of a correct peer
[**[TMBC-CorrFull]**][TMBC-CorrFull-link] in *peerIDs* at the time
*term - TD*, then if *FastSync* terminates successfully, it is at
some height *terminationHeight >= maxh*.

> We use *term - TD* as reference time, as we have to account
> for communication delay between the peer and *FS*. After the peer sent
> the last message to *FS*, the peer and *FS* run concurrently and
> independently. There is no assumption on the rate at which a peer can
> add blocks (e.g., it might be in the process of catching up itself).

>  (*TD* might depend on timeouts etc. We suggest that an acceptable
>  value for *TD* is in the range of apporx. 10 sec., that is the
>  intervall between two calls `QueryStatus()`; see below.


> Under [FS-ALL-CORR-PEER], if *peerIDs* contains a full node that is
> "synchronized with the blockchain", and *blockchainheight* is the height
> of the blockchain at time *term*, then  *terminationHeight* may even
> achieve
> blockchainheight - TD / ETIME*;
> cf. [**[TMBC-SEQ-APPEND-E]**][TMBC-SEQ-APPEND-E-link]. 





#### Liveness

<!--
> liveness specifications in English. Possibly with timing/fairness requirements:
e.g., if the component is connected to a correct full node and communication is
reliable and timely, then something good happens eventually. 
---->

#### **[FS-VC-ALL-CORR-TERM]**:
Under [FS-ALL-CORR-PEER], *Fastsync* eventually terminates successfully.


<!--
> How is the problem statement linked to the "Sequential Problem statement". 
Simulation, implementation, etc. relations 
---->


> We observe that all specifications that impose successful
> termination at an acceptable height are all conditional under
> [FS-ALL-CORR-PEER], that is, large parts of the current
> implementations of FastSync (V2) are not fault-tolerant. We will
> discuss this, and suggestions how to solve this after the
> description of the current protocol.

## Definitions

<!--
> In this section we become more concrete, with basic (abstracted) data types 
---->
<!--
> some math that allows to write specifications and pseudo code solution below.
Some variables, etc. 
---->

> We now introduce variables and auxiliary functions used by the protocol.

#### Inputs
- *startBlock*: the block Fastsync starts from
- *startState*: application state corresponding to *startBlock.Height*

#### **[FS-A-V2-INIT]**:
- *startBlock* is from the blockchain
- *startState* is the application state of the blockchain at Height *startBlock.Height*.

#### Variables
- *height*: initially *startBlock.Height + 1*
  > height should be thought of the "height of the next block we need to download"
- *state*: initially *startState*
- *peerIDs*: peer addresses [FS-A-PEER-IDS](#fs-a-peer-ids)
- *peerHeights*: stores for each peer the height it reported. initially 0
- *pendingBlocks*: stores for each height which peer was
  queried. initially nil for each height
- *receivedBlocks*: stores for each height which peer returned
  it. initially nil
- *blockstore*: stores for each height greater than
    *startBlock.Height*, the block of that height. initially nil for
    all heights
- *peerTimeStamp*: stores for each peer the last time a block was received
- *peerRate*: stores for each peer the rate of received data in Bytes/second

#### Auxiliary Functions

#### **[FS-FUNC-TARGET]**:
- *TargetHeight = max {peerHeigts(addr): addr in peerIDs} union {height}*

#### **[FS-FUNC-MATCH]**:


```go
func VerifyCommit(b Block, c Commit) Boolean
```
- Comment
    - Corresponds to `verifyCommit(chainID string, blockID
     types.BlockID, height int64, commit *types.Commit) error` in the
     current Golang implementation, which expects blockID and height 
	 (from the first block) and the
     corresponding commit from the following block. We use the
     simplified form for ease in presentation.

- Implementation remark
    <!-- - implements the check from -->
    <!--  [**[TMBC-SOUND-DISTR-PossCommit]**][TMBC-SOUND-DISTR-PossCommit--link], -->
    <!--  that is, that  *c* is a valid commit for block *b* -->
	- implements the check that  *c* is a valid commit for block *b*
- Expected precondition
    - *c* is a valid commit for block *b*
- Expected postcondition
    - *true* if precondition holds
	- *false* if precondition is violated
- Error condition
    - none
----

## Algorithm Invariants

> In contrast to the temporal properties above that define the problem
> statement, the following are invariants on the solution to the
> problem, that is on the algorithm. These invariants are useful for
> the verification, but can also guide the implementation.

#### **[FS-VAR-STATE-INV]**:
It is always the case that *state* corresponds to the application state of the
blockchain of that height, that is, *state = chain[height -
1].AppState*; *chain* is defined in
[**[TMBC-SEQ]**][TMBC-SEQ-link].

#### **[FS-VAR-PEER-INV]**:
It is always the case that the set *peerIDs* only contains nodes that
have not yet misbehaved (by sending wrong data or timing out).

#### **[FS-VAR-BLOCK-INV]**:
For *startBlock.Height <= i < height - 1*, let *b(i)* be the block with
height *i* in *blockstore*, it always holds that
*VerifyCommit(b(i), b(i+1).Commit) = true*. This means that *height*
can only be incremented if all blocks with lower height have been verified.


## FastSync V2

<!--
> Basic data structures. Simplified, so that we can focus on the distributed
algorithm here. If existing: link to Tendermint data structures, and mentioned
if details were omitted. 
---->

### Outline

<!--
> Describe solution (in English), decomposition into functions, where communication to other components happens.
---->

The protocol is described in terms of functions that are triggered by
(external) events. The implementation uses a scheduler and a
de-multiplexer to deal with communicating with peers and to
trigger the execution of these functions:

- `QueryStatus()`: regularly (currently every 10sec; necessarily
  interval greater than *2 Delta*) queries all peers from *peerIDs*
  for their current height [TMBC-CorrFull]. It does so
  by calling `Status(n)` remotely on all peers *n*.
  
- `CreateRequest`: regularly checks whether certain blocks have no
  open request. If a block does not have an open request, it requests
  one from a peer. It does so by calling `Block(n,h)` remotely on one
  peer *n* for a missing height *h*.
  
> We have left the strategy how peers are selected unspecified, and
> the currently existing different implementations of Fastsync differ
> in this aspect. 

The functions `Status` and `Block` are called by asynchronous
RPC. When they return, the following functions are called:

- `OnStatusResponse(addr Address, height int64)`: The full node with
  address *addr* returns its current height. The function updates the height
  information about *addr*, and may also increase *TargetHeight*.
  
- `OnBlockResponse(addr Address, b Block)`. The full node with
  address *addr* returns a block. It is added to *blockstore*. Then
  the auxiliary function `Execute` is called.

- `Execute()`: Iterates over the *blockstore*.  Checks soundness of
  the blocks, and
  executes the transactions of a sound block and updates *state*. 

> In addition to the functions above, the following two features are
> implemented in Fastsync V2

#### **[FS-V2-PEER-REMOVE]**:
Periodically, *peerTimeStamp* and *peerRate* are analyzed. If a peer *p*
has not provided a block recently (check of *peerTimeStamp[p]*) or it
has not provided sufficiently many data (check of *peerRate[p]*), then
*p* is removed from *peerIDs*.
  
#### **[FS-V2-TIMEOUT]**:

*Fastsync V2* starts a timeout whenever a block is
executed (that is, when the height is incremented). If the timeout expires
before the next block is executed, *Fastsync* terminates.
We say that if *peerIDs* is empty upon termination, then *Fastsync* terminates
with failure, otherwise it terminates successfully.




### Details

<!--
> Function signatures followed by pseudocode (optional) and a list of features (required):
> - Implementation remarks (optional)
>   - e.g. (local/remote) function called in the body of this function
> - Expected precondition
> - Expected postcondition
> - Error condition
---->



```go
func QueryStatus()
```
- Expected precondition
    - peerIDs initialized and non-empty
- Expected postcondition
    - call asynchronously `Status(n)` at each peer *n* in *peerIDs*.
- Error condition
    - fails if precondition is violated
----

```go
func OnStatusResponse(addr Address, ht int64)
```
- Comment
    - *ht* is a height
	- peers can provide the status without being called
- Expected precondition
    - *peerHeights(addr) <= ht*
- Expected postcondition
    - *peerHeights(addr) = ht*
	- *TargetHeight* is updated
- Error condition
    - if precondition is violated: *addr* not in *peerIDs* (that is,
      *addr* is removed from *peerIDs*)
- Timeout condition
    - if `OnStatusResponse(addr, ht)` was not invoked within *2 Delta* after
	`Status(addr)` was called:  *addr* not in *peerIDs*
----


```go
func CreateRequest
```
- Expected precondition
    - *height < TargetHeight*
	- *peerIDs* nonempty
- Expected postcondition
    - Function `Block` is called remotely at a peer *addr* in peerIDs 
	  for a missing height *h*  
	  *Remark:* different implementations may have different
      strategies to balance the load over the peers
    - *pendingblocks(h) = addr*
----


```go
func OnBlockResponse(addr Address, b Block)
```
- Comment
    - if after adding block *b*, blocks of heights *height* and
      *height + 1* are in *blockstore*, then `Execute` is called
- Expected precondition
    - *pendingblocks(b.Height) = addr*
	- *b* satisfies basic soundness  
- Expected postcondition
    - if function `Execute` has been executed without error or was not executed:
        - *receivedBlocks(b.Height) = addr*
	    - *blockstore(b.Height) = b*
		- *peerTimeStamp[addr]* is set to a time between invocation and
          return of the function.
		- *peerRate[addr]* is updated according to size of received
          block
- Error condition
    - if precondition is violated: *addr* not in *peerIDs*; reset
	*pendingblocks(b.Height)* to nil;
- Timeout condition
    - if `OnBlockResponse(addr, b)` was not invoked within *2 Delta* after
	`Block(addr,h)` was called for *b.Height = h*: *addr* not in *peerIDs*
----



```go
func Execute()
```
- Comments
    - none
- Expected precondition
	- application state is the one of the blockchain at height
      *height - 1*
	- **[FS-V2-Verif]** for any two blocks *a* and *b* from
	*receivedBlocks*: if
	  *a.Height + 1 = b.Height* then *VerifyCommit (a,b.Commit) = true*
- Expected postcondition
    - Any two blocks *a* and *b* violating [FS-V2-Verif]:
	  *a* and *b* not in *blockstore*; nodes with Address 
	  receivedBlocks(a.Height) and receivedBlocks(b.Height) not in peerIDs
	- height is updated height of complete prefix that matches the blockchain
	- state is the one of the blockchain at height *height - 1*
- Error condition
    - none
----

## Analysis of Fastsync V2

####  **[FS-ISSUE-KILL]**:
If two blocks are not matching [FS-V2-Verif], `Execute` dismisses both
blocks and removes the peers that provided these blocks from
*peerIDs*. If block *a* was correct and provided by a correct peer *p*,
and block b was faulty and provided by a faulty peer, the protocol
- removes the correct peer *p*, although it might be useful to
  download blocks from it in the future
- removes the block *a*, so that a fresh copy of *a* needs to be downloaded
  again from another peer
  
By [FS-A-PEER] we do not put a restriction on the number
  of faulty peers, so that faulty peers can make *FS* to remove all
  correct peers from *peerIDs*. As a result, this version of
  *Fastsync* violates [FS-VC-CORR-INV-SYNC].


####  **[FS-ISSUE-NON-TERM]**:

Due to [**[FS-ISSUE-KILL]**](#fs-issue-kill), from some point on, only
faulty peers may be in *peerIDs*. They can thus control at which rate
*Fastsync* gets blocks. If the timeout duration from [FS-V2-TIMEOUT]
is greater than the time it takes to add a block to the blockchain
(LTIME in [**[TMBC-SEQ-APPEND-E]**][TMBC-SEQ-APPEND-E-link]), the
protocol may never terminate and thus violate [FS-VC-TERM].  This
scenario is even possible if a correct peer is always in *peerIDs*,
but faulty peers are regularly asked for blocks.

### Consequence

The issues [FS-ISSUE-KILL] and [FS-ISSUE-NON-TERM] explain why the
temporal properties that are relevant for termination, namely,
[FS-VC-ALL-CORR-TERM] and [FS-VC-ALL-CORR-NONABORT], need to be
restricted to the case where all peers are correct
[FS-ALL-CORR-PEER]. In a fault tolerance context this is problematic,
as it means that faulty peers can prevent *FastSync* from termination.

Similarly, [FS-VC-CORR-INV-SYNC] and [FS-VC-CORR-INV] needed to be
restricted to [FS-ALL-CORR-PEER]. Again, if faults would be considered,
this would imply that if *Fastsync* terminates, it cannot be
guaranteed that a "reasonable" target height will be reached.

# Part III - Suggestions for an Improved Fastsync Implementation 

### Temporal Properties

Instead of the limited termination properties [FS-VC-ALL-CORR-TERM]
and [FS-VC-ALL-CORR-NONABORT], a fault-tolerant solution shall satisfy
the following two properties:

#### **[NewFS-VC-NONABORT]**:
If there is one correct process in *peerIDs* [FS-SOME-CORR-PEER],
*Fastsync* never terminates with failure. (Together with [FS-VC-TERM] below that means
it will terminate successfully.)

#### **[NewFS-VC-TERM]**:
*Fastsync* eventually terminates (successfully or with failure).


### Solution for [FS-ISSUE-KILL]

To avoid [FS-ISSUE-KILL], we observe that
[**[TMBC-FM-2THIRDS]**][TMBC-FM-2THIRDS-link] ensures that from the
point a block was created, we assume that more than two thirds of the
validator nodes are correct until the *trustingPeriod* expires.  Under
this assumption, assume the trusting period of *startBlock* is not
expired by the time *FastSync* checks a block *b1* with height
*startBlock.Height + 1*. To do so, we first need to check whether the
Commit in the block *b2* with *startBlock.Height + 2* contains more
than 2/3 of the voting power in *startBlock.NextValidators*. If this
is the case we can check *VerifyCommit (b1,b2.Commit)*. If we perform
checks in this order we observe:
  - By assumption, *startBlock* is OK, 
  - If the first check (2/3 of voting power) fails,
    the peer that provided block *b2* is faulty, 
  - If the first check passes and the second check
    fails (*VerifyCommit*), then the peer that provided *b1* is
    faulty.
  - If both checks pass, the can trust *b1*

Based on this reasoning, we can ensure to only remove faulty peers
from *peerIDs*.  That is, if
we sequentially verify blocks starting with *startBlock*, we will
never remove a correct peer from *peerIDs* and we will be able to
ensure the following invariant:


#### **[NewFS-VAR-PEER-INV]**:
If a peer never misbehaves, it is never removed from *peerIDs*. It
follows that under [FS-SOME-CORR-PEER], *peerIDs* is always non-empty.

> To perform these checks, we suggest to change the protocol as follows


#### Fastsync has the following configuration parameters:
- *trustingPeriod*: a time duration; cf.
  [**[TMBC-TIME-PARAMS]**][TMBC-TIME-PARAMS-link].

> [NewFS-A-INIT] is the suggested replacement of [FS-A-V2-INIT]. This will
> allow us to use the established trust to understand precisely which
> peer reported an invalid block in order to ensure the
> invariant [NewFS-VAR-TRUST-INV] below:

#### **[NewFS-A-INIT]**:
- *startBlock* is from the blockchain, and within *trustingPeriod*
(possible with some extra margin to ensure termination before
*trustingPeriod* expired)
- *startState* is the application state of the blockchain at Height
  *startBlock.Height*.
- *startHeight = startBlock.Height*

#### Additional Variables
- *trustedBlockstore*: stores for each height greater than or equal to
    *startBlock.Height*, the block of that height. Initially it
    contains only *startBlock*


#### **[NewFS-VAR-TRUST-INV]**:
Let *b(i)* be the block in *trustedBlockstore*
with b(i).Height = i. It holds that
for *startHeight < i < height - 1*, 
*VerifyCommit (b(i),b(i+1).Commit) = true*.



> We propose to update the function `Execute`. To do so, we first
> define the following helper functions:

```go
func ValidCommit(VS ValidatorSet, C Commit) Boolean
```
- Comments
    - checks validator set based on [**[TMBC-FM-2THIRDS]**][TMBC-FM-2THIRDS-link]
- Expected precondition
	-  The validators in *C*
	     - are a subset of VS
		 - have more than 2/3 of the voting power in VS
- Expected postcondition
    - returns *true* if precondition holds, and *false* otherwise
- Error condition
    - none 
----


```go
func SequentialVerify {
	while (true) {
		b1 = blockstore[height];
		b2 = blockstore[height+1];
		if b1 == nil or b2 == nil {
			exit;
		}
		if ValidCommit(trustedBlockstore[height - 1].NextValidators, b2.commit) {
			// we trust b2
			if VerifyCommit(b1, b2.commit) {
				trustedBlockstore.Add(b1);
				height = height + 1;
			}
			else {
				// as we trust b2, b1 must be faulty
				blockstore.RemoveFromPeer(receivedBlocks[height]);
				// we remove all blocks received from the faulty peer
				peerIDs.Remove(receivedBlocks(bnew.Height));
				exit;
			
			}			
		} else {
			// b2 is faulty
			blockstore.RemoveFromPeer(receivedBlocks[height + 1]);
			// we remove all blocks received from the faulty peer
		    peerIDs.Remove(receivedBlocks(bnew.Height));
			exit;
			}
		}
}
```
- Comments
    - none
- Expected precondition
	- [NewFS-VAR-TRUST-INV]
- Expected postcondition
    - [NewFS-VAR-TRUST-INV]
	- there is no block *bnew* with *bnew.Height = height + 1* in
      *blockstore*
- Error condition
    - none
----


> Then `Execute` just consists in calling `SequentialVerify` and then
> updating the application state to the (new) height.

```go
func Execute()
```
- Comments
    - first `SequentialVerify` is executed
- Expected precondition
	- application state is the one of the blockchain at height
      *height - 1*
	- [NewFS-NOT-EXP] *trustedBlockstore[height-1].Time > now - trustingPeriod*
- Expected postcondition
    - there is no block *bnew* with *bnew.Height = height + 1* in
      *blockstore*
	- state is the one of the blockchain at height *height - 1*
	- if height = TargetHeight: **terminate successfully**
- Error condition
    - fails if [NewFS-NOT-EXP] is violated
----




### Solution for [FS-ISSUE-NON-TERM]

As discussed above, the advantageous termination requirement is the
combination of [NewFS-VC-NONABORT] and [NewFS-VC-TERM], that is, *Fastsync*
should terminate successfully in case there is at least one correct
peer in *peerIDs*. For this we have to ensure that faulty processes
cannot slow us down and provide blocks at a lower rate than the
blockchain may grow. To ensure that we will have to add an assumption
on message delays.

#### **[NewFS-A-DELTA]**:

*2 Delta < ETIME*; cf. [**[TMBC-SEQ-APPEND-E]**][TMBC-SEQ-APPEND-E-link].

> This assumption implies that the timeouts for `OnBlockResponse` and
> `OnStatusResponse` are such that a faulty peer that tries to respond
> slower than *2 Delta* will be removed. In the following we will
> provide a rough estimate on termination time in a fault-prone
> scenario.


> In the following
> we assume that during a "long enough" finite good period no new
> faulty peers are added to *peerIDs*. Below we will sketch how "long
> enough" can be estimated based on the timing assumption in this
> specification. 


#### **[NewFS-A-STATUS-INTERVAL]**:
Let Sigma be the (upper bound on the)
time between two calls of `QueryStatus()`.

#### **[NewFS-A-GOOD-PERIOD]**:

A time interval *[begin,end]* is *good period* if:

- *fmax* is the number of faulty peers in *peerIDs* at time *begin*
- *end >= begin + 2 Delta (fmax + 3)*
- no faulty peer is added before time *end*

> In the analysis below we assume that the termination condition of
> *Fastsync* V2 [FS-V2-TIMEOUT] (that is, termination when the height is not incremented for
> some time), is replaced by the termination condition
> *height = TargetHeight* in the postcondition of
> `Execute`. Therefore, [NewFS-A-STATUS-INTERVAL] does not interfere
> with in analysis. If a correct peer reports a new height "shortly
> before termination" this leads to an additional round trip to
> request and add the new block. Then [NewFS-A-DELTA] ensures that
> *Fastsync* catches up.

Arguments: 

1. If a faulty peer *p* reports a faulty block, `SequentialVerify` will
  eventually remove *p* from *peerIDs*
  
2. By `SequentialVerify`, if a faulty peer *p* reports multiple faulty
  blocks, *p* will be removed upon trying to check the block with the
  smallest height received from *p*.

3. Assume whenever a block does not have an open request, `CreateRequest` is
   called immediately, which calls `Block(n)` on a peer. Say this
   happens at time *t*. There are two cases: 
  
   - by t + 2 Delta a block is added to *blockStore*
   - at t + 2 Delta `Block(n)` timed out and *n* is removed from
       peer.
	   
4. Let *f(t)* be the number of faulty peers in *peerIDs* at time *t*;  
   *f(begin) = fmax*.
	   
5. Let t_i be the sequence of times `OnBlockResponse(addr,b)` is
   invoked or times out with *b.Height = height + 1*.
   
6. By 3., 
   - (a). *t_1 <= begin + 2 Delta* 
   - (b). *t_{i+1} <= t_i + 2 Delta* 

7. By an inductive argument we prove for *i > 0* that

   - (a). *height(t_{i+1}) > height(t_i)*, or
   - (b). *f(t_{i+1}) < f(t_i))* and *height(t_{i+1}) = height(t_i)*  

   Argument: if the peer is faulty and does not return a block, the
   peer is removed, if it is faulty and returns a faulty block
   `SequentialVerify` removes the peer (b). If the returned block is OK,
   height is increased (a).
	 
8. By 2. and 7., faulty peers can delay incrementing the height at
   most *fmax* times, where each time "costs" *2 Delta* seconds. We
   have additional *2 Delta* initial offset (3a) plus *2 Delta* to get
   all missing blocks after the last fault showed itself. (This
   assumes that an arbitrary number of blocks can be obtained and
   checked within one round-trip 2 Delta; which either needs
   conservative estimation of Delta, or a more refined analysis). Thus
   we reach the *targetHeight* and terminate by time *end*.


# References

<!--
> links to other specifications/ADRs this document refers to
---->


[[block]] Specification of the block data structure. 

<!-- [[blockchain]] The specification of the Tendermint blockchain. Tags refering to this specification are labeled [TMBC-*]. -->

[block]: https://github.com/tendermint/spec/blob/master/spec/blockchain/blockchain.md

<!-- [blockchain]: https://github.com/informalsystems/VDD/tree/master/blockchain/blockchain.md -->

[TMBC-HEADER-link]: #tmbc-header

[TMBC-SEQ-link]: #tmbc-seq

[TMBC-CorrFull-link]: #tmbc-corrfull

[TMBC-CORRECT-link]: #tmbc-correct

[TMBC-Sign-link]: #tmbc-sign

[TMBC-FaultyFull-link]: #tmbc-faultyfull

[TMBC-TIME-PARAMS-link]: #tmbc-time-params

[TMBC-SEQ-APPEND-E-link]: #tmbc-seq-append-e

[TMBC-FM-2THIRDS-link]: #tmbc-fm-2thirds

[TMBC-Auth-Byz-link]: #tmbc-auth-byz

[TMBC-INV-SIGN-link]: #tmbc-inv-sign

[TMBC-SOUND-DISTR-PossCommit--link]: #tmbc-sound-distr-posscommit

[TMBC-INV-VALID-link]: #tmbc-inv-valid

<!-- [TMBC-HEADER-link]: https://github.com/informalsystems/VDD/tree/master/blockchain/blockchain.md#tmbc-header -->

<!-- [TMBC-SEQ-link]: https://github.com/informalsystems/VDD/tree/master/blockchain/blockchain.md#tmbc-seq -->

<!-- [TMBC-CorrFull-link]: https://github.com/informalsystems/VDD/tree/master/blockchain/blockchain.md#tmbc-corrfull -->

<!-- [TMBC-Sign-link]: https://github.com/informalsystems/VDD/tree/master/blockchain/blockchain.md#tmbc-sign -->

<!-- [TMBC-FaultyFull-link]: https://github.com/informalsystems/VDD/tree/master/blockchain/blockchain.md#tmbc-faultyfull -->

<!-- [TMBC-TIME-PARAMS-link]: https://github.com/informalsystems/VDD/tree/master/blockchain/blockchain.md#tmbc-time-params -->

<!-- [TMBC-SEQ-APPEND-E-link]: https://github.com/informalsystems/VDD/tree/master/blockchain/blockchain.md#tmbc-seq-append-e -->

<!-- [TMBC-FM-2THIRDS-link]: https://github.com/informalsystems/VDD/tree/master/blockchain/blockchain.md#tmbc-fm-2thirds -->

<!-- [TMBC-Auth-Byz-link]: https://github.com/informalsystems/VDD/tree/master/blockchain/blockchain.md#tmbc-auth-byz -->

<!-- [TMBC-INV-SIGN-link]: https://github.com/informalsystems/VDD/tree/master/blockchain/blockchain.md#tmbc-inv-sign -->

<!-- [TMBC-SOUND-DISTR-PossCommit--link]: https://github.com/informalsystems/VDD/tree/master/blockchain/blockchain.md#tmbc-sound-distr-posscommit -->

<!-- [TMBC-INV-VALID-link]: https://github.com/informalsystems/VDD/tree/master/blockchain/blockchain.md#tmbc-inv-valid -->

[LCV-VC-LIVE-link]: https://github.com/informalsystems/VDD/tree/master/lightclient/verification.md#lcv-vc-live

[lightclient]: https://github.com/interchainio/tendermint-rs/blob/e2cb9aca0b95430fca2eac154edddc9588038982/docs/architecture/adr-002-lite-client.md

[failuredetector]: https://github.com/informalsystems/VDD/blob/master/liteclient/failuredetector.md

[fullnode]: https://github.com/tendermint/spec/blob/master/spec/blockchain/fullnode.md

[FN-LuckyCase-link]: https://github.com/tendermint/spec/blob/master/spec/blockchain/fullnode.md#fn-luckycase

[blockchain-validator-set]: https://github.com/tendermint/spec/blob/master/spec/blockchain/blockchain.md#data-structures

[fullnode-data-structures]: https://github.com/tendermint/spec/blob/master/spec/blockchain/fullnode.md#data-structures

[FN-ManifestFaulty-link]: https://github.com/tendermint/spec/blob/master/spec/blockchain/fullnode.md#fn-manifestfaulty
