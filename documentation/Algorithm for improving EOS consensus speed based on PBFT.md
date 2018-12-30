# Algorithm for improving EOS consensus speed based on PBFT

## 1 Background
#### 1.1 Phenomenon
There is a gap of 325+ blocks between the chain height of the main network and the consensus height, which is equivalent to a time difference of ~3 minutes. In other words, the current submitted trx needs to wait ~3 minutes to confirm whether it is recorded in the chain. This performance is unbearable for many DApps, especially those that require immediate confirmation.


#### 1.2 Reasons
* The reason for this phenomenon on the main network is that in EOS's DPOS-based consensus algorithm, all block synchronization and acknowledgment information can only be sent when it is out of the block. That is to say, in the case where the BP<sub>1</sub> is out of the block (the block is BLK) and the BP<sub>1</sub>~BP<sub>21</sub> round out block, BP<sub>2</sub>~BP<sub>21</sub> will receive and verify BLK one after another, but all BPs can only send confirmation messages to BLK when they are out of the block. This is also why we see the logs in the nodes. When each BP comes out of the first block in the schedule, `confirmed` is always 240. The fastest speed of DPOS+Pipeline BFT consensus (ie, the smallest gap between head and LIB) is 325.


* 240 = (21-1)*12
This is actually the sum of all the blocks in the previous round (in the case of good network conditions). Each node maintains a vector `confirm_count` with a length of up to 240 and an initial value of 14 in `block_header`, corresponding to all blocks received but not agreed upon and the required number of acknowledgments. Whenever more than one BP acknowledges the block, the value of the corresponding block is -1, until the required number of acknowledgments of a block is reduced to 0, the block and all previous blocks enter the consensus ([related code] (https: //github.com/EOSIO/eos/blob/905e7c85714aee4286fa180ce946f15ceb4ce73c/libraries/chain/block_header_state.cpp#L188)).

* 325 = 12*(13+1+13) + 1
The entire network needs 15 people to confirm to reach a consensus. Everyone by default confirms their own blocks, so each block requires 14 people's implicit confirm and (explicit) confirm. The 14th BP has confirmed that the number of people has reached 15 since it was included in the block, so it will issue implicit confirm and (explicit) confirm. Then ideally, after a block is generated from it, it will not be able to get the consensus of the whole network and enter the LIB until the first block produced by the 28th BP. Therefore, there are the above calculations.


* ** We believe that all BPs do not need to wait until the block to confirm other blocks, use PBFT** (Practical Byzantine Fault Tolerance<sup>[1]</sup>)** instead of Pipeline BFT, let The real-time confirmation of the blocks currently being produced between the BPs enables the entire system to eventually reach a near real-time consensus speed. **

## 2 Algorithm Core
* Retain the DPOS's BP Schedule mechanism and, like EOS, impose strong constraints on synchronized clock and BP Schedule.

* Remove the Pipeline BFT part of the consensus in EOS (ie remove the implicit confirm and (explict) confirm part of the original EOS block), because in extreme cases it may conflict with PBFT consensus results.

* Consensus communication mechanism uses existing p2p networks for communication, but increases communication costs, using the PBFT mechanism to broadcast prepare and commit information.

* Optimized by batch (replaces the requirement for consensus on each block in PBFT), a batch consensus can be reached to approximate the ideal state of real-time BFT and reduce network load.


## 3 Basic concepts
#### 3.1 The concrete realization of BP change in DPOS
* In the current code, the voting rank is refreshed every 60s (120 blocks) ([related code] (https://github.com/EOSIO/eos/blob/8f0f54cf0c15c4d08b60c7de4fd468c5a8b38f2f/contracts/eosio.system/producer_pay.cpp#L47) ), if the first 21 changes, will issue the `promoting proposed schedule` ([related code] at the next refresh ranking (https://github.com/EOSIO/eos/blob/8f0f54cf0c15c4d08b60c7de4fd468c5a8b38f2f/libraries/chain/controller. Cpp#L909))

* When a block containing `promoting proposed schedule` enters LIB, BP will update the `pending_schedule` in its own block header.

* After 2/3 +1 BP nodes have updated the block header, `pending schedule` has reached a consensus. BP will update the `active schedule` to the value of the `pending schedule` at this time, and start to block out according to the new BP combination. The whole process needs to go through at least two rounds of complete outbound.

* Every new BP combination must be able to reach a consensus to be effective. In other words, if 7 or more nodes in the network are unable to communicate properly, then no new BP can be generated by voting. The LIB of the network will stay at the consensus point where the node crashes.

* DPOS can effectively avoid some of the fork problems, so the DPOS consensus mechanism for BP elections will still be used. All BP changes will not take effect until the probability schedule enters LIB.

#### 3.2 Prerequisites for PBFT
* If there are f Byzantine nodes in the network, then the total number of nodes n is required to satisfy n≥3f+1. A Byzantine node is a node that is inconsistent in its external state, including a node with a main action and a node that fails or partially fails due to network reasons.

* All information is ultimately reachable: All communication information may be delayed/out of order/discarded, but retrying will ensure that the information will eventually be delivered.


#### 3.3 The key concepts in PBFT correspond to DPOS
**pre-prepare** means that the primary node receives the request and broadcasts it to all replicas in the network. It can be analogized to BP in DPOS and broadcast to the whole network.


**prepare** means that the replica will broadcast the request to the entire network after receiving the request. The analogy is DPOS. All nodes receive the block and verify the success of the broadcast of the received information.


**commit** means that the replica receives enough prepare messages for the same request to broadcast the request to the entire network. It can be analogized that the node in DPOS receives enough prepare messages for the same block, and proposes a proposed lib message.

**committed-local**, which means that the replica receives enough commit messages for the same request and completes the verification. It can be compared to the LIB promotion in DPOS.


**view change** means that the primary node loses the trust of the replica for various reasons, and the entire system changes the primary process. Since EOS adopts the DPOS algorithm, all BPs are determined in advance by voting. The order of the whole system is completely unchanged under the same BP schedule. When the network is in good condition and the BP schedule is unchanged, it can be considered. There is no view change.
After the introduction of PBFT, in order to avoid the fork caused the consensus does not advance, join the view change mechanism, discard all unconsented blocks for replay, and continually try again until the consensus is continued.

**checkpoint**, which refers to the record of consensus evidence at a block height to provide security proof. When enough checkpoints of the replica are the same, the checkpoint is considered to be stable. The checkpoint generation consists of two major categories. One is fixed k block generation; the other is a special point that needs to provide security proof, such as a block with a change in BP schedule.


## 4 Unoptimized version overview

the term:  
* v: view version
* i: BP's name
* BLK<sub>n</sub>: the nth block
* d<sub>n</sub>: Consensus message digest corresponding to the nth block digest
* σ<sub>i</sub>: signature of BP named i
* n: height of the block

All BPs agree on each block in order, using the PBFT mechanism. The following subsections describe:


#### **4.1 Under normal circumstances (no BP changes or forks, and the network is in good condition)**
The **pre-prepare** phase is no different from the current logic, that is, the BP broadcasts its signed block.


**prepare** stage, BP<sub>i</sub> receives the block BPK<sub>n</sub> of the current BP signature, and is verified (PREPARE,v,n,d<sub>n< /sub>,i)<sub>σ<sub>i</sub></sub> message, waiting for consensus. When BP<sub>i</sub> receives a 2/3 node and sends a PREPARE message to BLK<sub>n</sub> under view v, it is considered that the peer of the block has reached a consensus in the network. The PREPARE message has been sent and cannot be changed.


**commit** stage, when BLK<sub>n</sub> is marked as ***prepared***, BP<sub>i</sub> is issued (COMMIT,v,n,d<sub>n </sub>,i)<sub>σ<sub>i</sub></sub>. It should be noted that PBFT implements security by guaranteeing strict order, so the consensus on all nodes is also strictly in order, that is, (PREPARE, v, n, d<sub>n</ Sub>,i)<sub>σ<sub>i</sub></sub> is issued under the premise that BLK<sub>n-1</sub> is at least ***committed under the same view *** Status.


**LIB promotion under the whole network perspective**, when BP<sub>i</sub> receives 2/3 nodes, it sends a COMIT message to BLK<sub>n</sub>, BP<sub>i </sub> believes that there is a consensus on the commit of this block in the network, that is, the block has reached a consensus, this block is marked as *committed* state, and the LIB is raised to the current height n, and then the next block is prepared. If the height of the block is H<sub>i</sub>, the LIB heights of all BPs are arranged in descending order to obtain a vector V<sub>c</sub> of length L, from the perspective of the whole network, V<sub >c</sub>[2/3L] and below LIB can be considered as ***stable***, V<sub>c</sub>[2/3L] is the LIB height of the whole network at this time.


** For the same block, only the sufficient PREPARE message is collected before entering the commit phase. Similarly, only after collecting enough COMMIT messages, will start the prepare for the next block, otherwise it will be resent until the number of messages meets the requirements or the view change (see below). **

#### **4.2 When BP changes, **
**pre-prepare** phase, no difference from 4.1.


In the **prepare** and **commit** phases, due to the timing of the consensus between different BPs on the information of BP changes, there will be an inconsistent state between the BPs for the schedule.
Taking BP<sub>i</sub> as an example, BP<sub>i</sub> receives the block BLK<sub>n</sub> of the current BP<sub>c</sub> signature, if Most BP's active schedule has been changed to S', and BP<sub>i</sub> is still S, then BP<sub>i</sub> will continue to wait for the PREPARE information sent by BP in S, which cannot Enter the commit phase.
However, at this time, most nodes in the network will still reach a consensus with each other, resulting in an increase in the LIB of the entire network. If BP<sub>i</sub> receives enough commit information under the same view, BP<sub>i</sub> enters the commit-local state and promotes its own LIB.



#### **4.3 When a fork is generated**
**pre-prepare** phase, no difference from 4.1.

**prepare** and **commit** phases, when BP<sub>i</sub> does not collect enough PREPARE or COMMIT messages in timeout=T, that is, the consensus is not promoted during this time period, The VIEW-CHANGE message initiates a view change and no longer receives any messages except VIEW-CHANGE, NEW-VIEW, and CHECKPOINT.

**view change** stage, BP<sub>i</sub> is issued (VIEW-CHANGE, v+1, h<sub>lib</sub>, n, i)<sub>σ<sub>i< /sub></sub> message. When 2/3 +1 v'=v+1 VIEW-CHANGE messages are collected, they are sent by the next BP in the schedule (NEW-VIEW, v+1, n, VC, O)<sub>σ< Sub>bp</sub></sub> message, where VC is all VIEW-CHANGE messages including BP signatures, and O is all unrecognized PRE-PREPARE messages (between h<sub>lib</sub> and Between n<sub>max</sub>). After other BPs receive and verify that the NEW-VIEW message is valid, discard all blocks that are not currently agreed, and re-prepone the prepare and commit phases based on all PRE-PREPARE messages.
If the view change fails to reach a consensus within timeout=T (there is no correct NEW-VIEW message sent), a new round of v+2 view change is initiated, the wait time is timeout=2T, and so on, until the network status Convergence, the consensus began to improve.


**Remarks**: The original PBFT does not have a fork problem, because PBFT will only process the next request after a request has reached a consensus.


## 5 Problems with unoptimized versions:
#### 5.1 Consensus speed
When the consensus speed of a block is less than 500ms, that is, the transmission of two rounds of messages can receive enough confirmation numbers within 500ms, the gap between head and LIB can be stabilized and can approach 1 block, that is, real-time consensus. When the average consensus speed for a block is greater than or equal to 500ms or the network state is extremely poor, resulting in too many retries, the algorithm may be slower than DPOS+Pipeline BFT.

#### 5.2 Network overhead
Suppose the node in the network is N, the message propagation uses the gossip algorithm, and the block size is B. Then the message that DPOS needs to propagate is N<sup>2</sup>, and the required bandwidth is BN<sup>2</sup>.
Assuming that the PREPARE and COMMIT message sizes are p and c, respectively, the number of messages that PBFT+DPOS needs to propagate is (1+r<sub>p</sub>+r<sub>c</sub>)N<sup>2 </sup>, where 1 is the pre-prepare transmission, r<sub>m</sub>, r<sub>c</sub> is the number of retries for prepare and commit, and the required bandwidth is (B+pr <sub>p</sub>+cr<sub>c</sub>)N<sup>2</sup>. When the p and c optimizations are small enough, the extra bandwidth overhead depends mainly on the number of retries.

## 6 Overview of the optimized version
#### 6.1 Implement batch consensus through adaptive granularity adjustment
##### 6.1.1 batch strategy
The height of LIB is h<sub>LIB</sub>
The height of the block at the highest point in the fork is h<sub>HEAD</sub>
The block height involved in the BP schedule change is h<sub>s</sub>
Batch consensus batch:
* batch<sub>min</sub> = 1
* batch<sub>max</sub> = min(default_batch_max, h<sub>HEAD</sub> - h<sub>LIB</sub>)

When batch<sub>max</sub> does not contain BP Schedule changes, batch = batch<sub>max</sub>
Batch = h<sub>s</sub> - 1 when batch<sub>max</sub> contains BP Schedule changes and h<sub>LIB</sub> < h<sub>s</sub>
When batch<sub>max</sub> contains BP Schedule changes and h<sub>LIB</sub> == h<sub>s</sub>, batch = batch<sub>max</sub>

##### 6.1.2 Batch Consensus Principle
When there is no fork condition, the above construction can be compared with the consensus in the view of PBFT. And based on the basic structure of Merkle Tree, when most nodes can reach consensus on BLKn Hash, all previous blocks should be consensus. The total order of the block is guaranteed here.

When there is a fork condition, the PREPARE information cannot be changed, otherwise it may appear as a Byzantine error. At this point, it is necessary to continuously resend the current PREPARE message until the network reaches a consensus or triggers a timeout to initiate a view change.

##### 6.1.3 Implementation Method
* Whenever a new block is received, BP generates PREPARE information through the batch policy for caching and broadcasting.

* Each BP maintains a minimum water level h for the block_header, and the highest water level H, corresponding to the lowest point and the highest point that have not yet reached consensus.

* Maintain two vectors of length (H-h) V<sub>p</sub> & V<sub>c</sub> at the same time, including the number of PREPARE messages and COMMIT messages required for each block between water levels.

* Each time a PREPARE message (or COMMIT message) of height n is received, it is verified by the signature and digest of the message and confirmed that he is in the same fork as himself, and then V<sub>p</sub>(V< All values ​​in the sub>c</sub>) (h ≤ n) -1.

* Constantly resend the PREPARE message (or COMMIT message) with the height H on the same fork until the consensus is reached or the timeout triggers View Change (based on New View to restart the PBFT consensus, v' = v+1).

* When a block at height x (h ≤ x ≤ H) collects more than 2/3 +1 PREPARE messages, the block contents from h to x are sequentially executed and all blocks of (h ≤ x) are marked as *prepared* And then automatically issue a COMIT message of height x.

* When a block at height y (y ≤ H) collects more than 2/3 +1 COMMIT messages, the block contents from h to y are sequentially executed and all blocks of (h ≤ y) are marked as *committed*. At this point, all blocks that think ≤y have reached a consensus and raise their LIB height to y.

* Generate checkpoints every few blocks to improve performance. When the latest checkpoint of more than 2/3 +1 in the network reaches a certain height c and is on the same fork, the checkpoint is considered stable.

##### 6.1.4 view change strategy
* BP becomes the backup of the previous person according to the schedule of the block, ensuring that only one person can be found in the primary after each view change.

* When the network begins to enter the view change, NEW-VIEW should re-consider the block between the lowest point h and the highest point H seen by 2/3 +1 people.

* The BP that issued NEW-VIEW should include all VIEW-CHANGE messages in the message, and calculate h and H based on all VIEW-CHANGE messages, and exceed (2/3 +1) in the [h, H] interval. The fork of the person's choice is issued.

* When BP receives the NEW-VIEW message and verifies it, it repeats the prepare based on the content of NEW-VIEW.

* If the view change cannot be completed within timeout=T, a new round of view change of v+2 will be initiated until the network has reached a consensus on the choice of fork.

#### 6.2 Avoid fork risk by always preparing the longest chain and combining view change
* When BP receives multiple forks, it should make a preference for the longest chain that can be seen at present, taking the longest-live-fork principle.

* BP should stagger the time point of BP switching when making a prepare, so as to avoid choosing a fork supported by a few people.

* **BP once you make a prepare for a fork, you can no longer change the prepare message, otherwise it may become a Byzantine error, ** BP needs to:
1) Constantly resend the previous PREPARE message, waiting for the final consensus. Even if this fork is not the longest chain, because there are more people to support, you should choose this fork;
2) Or wait for timeout=T, initiate a view change, all BP starts a new BPFT consensus based on the fork issued by NEW-VIEW;
3) Receive a COMMIT message or checkpoint that exceeds (2/3 +1) the same fork, discarding the current state synchronization to the height reached by the majority.

#### 6.3 Implementing GC and improving synchronization performance through Checkpoint mechanism
* BP continuously broadcasts its current checkpoint status within the network and receives checkpoints from other people.

* When there are more than (2/3 +1) people on the same branch whose checkpoint is higher than c, it is considered that CHECKPOINT<sub>c</sub> has been stable, and all caches such as PREPARE and COMMIT messages with a height lower than c are deleted.

* By verifying the correctness of the checkpoint, you can greatly improve the synchronization speed of the node.

## 7 FAQ
DPOS related issues (see 1.2)

1. Briefly explain how DPOS works.
Temporary
2. Why DPOS lib is 12 12 ups
Temporary
3. Why is the gap between HEAD and LIB of DPOS so large?
Temporary
4. How does DPOS work when BP changes?
Temporary
5. How is the data between nodes currently synchronized?
Temporary

PBFT related issues

1. Brief description of PBFT
Temporary

DPOS-PBFT related issues

1. Briefly explain how DPOS-PBFT works
See 5


2. Why can't I only broadcast the information of prepare once?
When there is a fork (or BP change) in the network, if there is only PREPARE information, all nodes cannot respond to the view change of other nodes, which will lead to hard fork. For example: Due to the nature of distributed networks, information can be delayed or disrupted. Suppose now that there are three consecutive blocks of BP A, B, C. If B does not receive the last block of A, then he will continue to block from the second to last block. This causes two forks to choose F1 F2. Assuming that the last block of A contains information about BP changes (the block is in F1), then the node that selected F1 needs a new BP S1 for consensus, and F2 The node needs the original BP S2 for consensus. The consensus group has changed, and it is very likely that both sides will eventually enter a consensus state, which will lead to the overall network bifurcation.


3. How does the prepare and commit retransmission mechanism work?
When a given acknowledgment is not received for a block that is *prepared* or *committed* after a given timeout T, the same message is retransmitted one more time until enough acknowledgments are collected or A view change has occurred.


4. Is there a fork risk when the BP set changes?
See 4.2


6. Do you need to wait for the consensus to complete before you can continue to block
The block can continue, and the consensus only affects the height of the LIB.


7. If the Nth block does not meet the BFT consensus number, but the N+1th block receives enough confirmation, what should I do?
For the optimized algorithm, you can start collecting consensus messages based on N+2 blocks.


8. Will continuous spurs be forked because the consensus is not quickly reached?
No, at least the status of DPOS will eventually lead to consensus on the longest chain.


9. Does the BFT commit information need to be written to the block?
All messages (issued and received) are only local. But they need to be kept for a period of time to provide evidence for peers.


10. What is the extra overhead?
See 5.2


11. Can the speed of consensus really improve? If the BFT consensus average time is >500ms, the height of BFT is lower than DPOS.
See 5.1


## 8 Reference
[1] http://pmg.csail.mit.edu/papers/osdi99.pdf