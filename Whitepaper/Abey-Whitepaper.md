# Abey Whitepaper

## Table of Contents
- [Abstract](#abstract)
- [1. Introduction](#1-introduction)
- [2. Background](#2-background)
  - [2.1 Related Works](#21-related-works)
- [3. Consensus](#3-consensus)
  - [3.1 Design Overview](#31-design-overview)
  - [3.2 Recap of DPoS Consensus Protocol](#32-recap-of-dpos-consensus-protocol)
    - [3.2.1 Daily Off-chain Consensus Protocol](#321-daily-off-chain-consensus-protocol)
    - [3.2.2 Mempool Subprotocol](#322-mempool-subprotocol)
    - [3.2.3 Main Consensus Protocol](#323-main-consensus-protocol)
  - [3.3 Variant Day Length and Committee Election](#33-variant-day-length-and-committee-election)
  - [3.4 Application-Specific Design](#34-application-specific-design)
    - [3.4.1 Physical Timing Restriction](#341-physical-timing-restriction)
  - [3.5 Computation and Data Sharding, and Speculative Transaction Execution](#35-computation-and-data-sharding-and-speculative-transaction-execution)
- [4. Smart Contracts in Virtual Machines](#4-smart-contracts-in-virtual-machines)
  - [4.1 Design Rationale](#41-design-rationale)
  - [4.2 Abey Virtual Machine (AVM)](#42-abey-virtual-machine-avm)
- [5. Blocks, State, and Transactions](#5-blocks-state-and-transactions)
- [6. Incentive Design](#6-incentive-design)
  - [6.1 Gas Fee and Sharding](#61-gas-fee-and-sharding)
- [7. Future Direction](#7-future-direction)
- [8. Conclusions](#8-conclusions)
- [References](#references)

---

## Abstract

In this paper, we present the initial design of **Abey (3.0+)** and other technical details. Briefly, our consensus design enjoys the same consistency, liveness, transaction finality, and security guarantees. We discuss optimizations, such as the frequency of rotating committee members and physical timestamp restrictions. The primary focus of Abey is to advance these concepts and build a blockchain specifically designed for the Abey ecosystem. We also utilize:

1. Data sharding and speculative transactions.  
2. Evaluation of smart contracts in a hybrid cloud infrastructure.  
3. Usage of existing volunteer computing protocols for a compensation infrastructure.  

---

## 1. Introduction

With the surging popularity of cryptocurrencies, blockchain technology has caught the attention of both industry and academia. One can think of blockchain as a shared computing environment where peers join and quit freely, with a commonly agreed consensus protocol. The decentralized nature of blockchain, together with transaction transparency, autonomy, and immutability, is critical to cryptocurrencies.

However, earlier-designed cryptocurrencies, such as Bitcoin [[1]](#ref-1) and Ethereum [[2]](#ref-2), have been recognized as unscalable in terms of transaction rates and not economically viable due to their energy consumption and computational power requirements.

With the growing demand for apps and platforms utilizing public blockchains, a viable protocol that enables higher transaction rates is the primary focus of new systems. For example, a public blockchain hosting computationally intensive peer-to-peer gaming applications with a large user base would face delays in transaction confirmation if it also hosted ICO contracts.

Other models, such as Delegated Proof of Stake (PoS) and Hybrid Consensus, also exist. Abey 2.0 adopted Hybrid Consensus, incorporating PBFT [[3]](#ref-3) and Proof of Work (PoW). Abey 3.0 phases out PoW for a more energy-efficient Proof-of-Stake protocol. PoS ensures safety as long as no more than one-third of validators are malicious. Validators achieve consensus using a modified PBFT [[4]](#ref-4), ensuring incentivization, instant finality, high throughput, and fair committee rotation.

---

## 2. Background

The core strength of this proposal lies in recognizing DPoS principles and utilizing DailyBFT rotating committees for validator fairness.

### 2.1 Related Works

In PoS systems, the native token stores both value and voting power. PoS achieves Sybil resistance through BFT while consuming less energy. Unlike PoW, PoS selects validators proportionally to staked holdings, reducing computation costs while offering scalability and security.

---
## 3. Consensus

### 3.1 Design Overview

Our consensus design is a proof-of-stake model that has been extended to meet the requirements of applications that demand high throughput and fairness. The protocol builds on Hybrid Consensus [[5]](#ref-5). Our adversary model follows Pass and Shi [[6]](#ref-6), where adversaries can adaptively corrupt nodes with some delay. Future work will formalize these modifications under the Universal Composability framework [[7]](#ref-7).

### 3.2 Recap of DPoS Consensus Protocol

#### 3.2.1 Daily Off-chain Consensus Protocol

In **DailyBFT**, committee members execute an **offchain Byzantine Fault Tolerant (BFT) instance** to establish a daily transaction log, while non-members verify progress by counting signatures from the committee. This design extends security guarantees to both non-members and late-joining nodes. It enforces a **termination agreement**, requiring that all honest participants converge on the same final log once the protocol terminates.  

Committee members produce **signed daily log hashes**, which are consumed by the Proof-of-Stake consensus protocol. These signed hashes guarantee **completeness** (no valid transaction is omitted) and **unforgeability** (signatures cannot be fabricated).  

During **key generation**, each node’s public key is added to a maintained key list. Upon receiving a **committee signal**, nodes may be conditionally elected as committee members. The system selectively opens participation to the committee through this controlled election mechanism.  

- **If the node is a BFT committee member:**  
  A **BFT virtual node** is instantiated, denoted as *BFTpk*, which begins receiving transactions (*TXs*). The log’s completion status is continuously monitored, and the process halts once a **stop signal** is signed by at least one-third of the initially designated committee public keys. Throughout execution, the protocol performs a recurring *“Until Done”* check: after each gossip round completes, all stop log entries are removed, ensuring freshness and consistency.  

- **If the node is not a BFT committee member:**  
  Upon receiving a transaction, the node appends it to its local history and verifies that the transaction has been signed by at least one-third of the distinct committee public keys. This guarantees the correctness and validity of the received log.  

Finally, to prevent **namespace collisions**, the signing algorithm uses unique prefixes:  
- `"0"` for messages belonging to the inner BFT instance.  
- `"1"` for messages belonging to the outer DailyBFT layer.  

#### 3.2.2 Mempool Subprotocol

The mempool maintains a set of pending transactions. When a new transaction arrives, it is initialized with status zero and added to the set. On receipt of a proposal, transactions are grouped into blocks and disseminated by gossip. Confirmed transactions are removed. Query interfaces allow validation of transaction state.

#### 3.2.3 Main Consensus Protocol

A new node inherits transcript history and implicit routing from peers. It interacts with mempools, preprocessing functions, the off-chain DailyBFT consensus, and on-chain validation.

### 3.3 Variant Day Length and Committee Election

Committees rotate after fixed time periods, with the chain serving as a logical clock [[8]](#ref-8). Committee members are selected as miners of the most recent blocks (of size `csize`). Reducing rotation frequency minimizes switching overhead but enforced switches guarantee fairness. If misbehavior is detected, authenticated evidence is submitted on-chain, triggering a forced switch. This approach is inspired by Thunderella [[9]](#ref-9).

### 3.4 Application-Specific Design

#### 3.4.1 Physical Timing Restriction

In conventional consensus designs, miners, committee members, or leaders are often able to **reorder transactions** within a short timing window. While this flexibility may not affect some applications, it creates a critical fairness problem for decentralized systems such as **commercial exchanges**, where the exact timing of trades must be preserved. If reordering is permitted, malicious or even rational participants may reorder or insert their own transactions to gain unfair profit. This incentive becomes even stronger under conditions of **high throughput**.  

A further complication arises because malicious reordering is difficult to detect: natural **network latency** already causes reordering, and latency values are only observable by the receiving node. Thus, nodes can always claim latency-related delays, making it impossible to prove malicious intent.  

To mitigate these problems—particularly for use cases like decentralized advertisement exchanges—Abey blockchain introduces a mechanism called the **sticky timestamp**. With this approach, a client must attach a physical timestamp (*Tp*) to each transaction at the moment it is created. This timestamp, along with other transaction data, is digitally signed. Validators then enforce additional checks using a heuristic parameter (*TΔ*) to ensure the validity of the timestamp. The procedure is shown in **Algorithm 1**.  

During log materialization within the BFT process, the leader orders the batch of transactions primarily by their physical timestamps, using sequence numbers to break ties (a rare case). While this ordering step could technically be deferred until evaluation and verification, performing it early simplifies the workflow.  

This modification introduces several important properties:  

1. **Strict intra-node ordering:** The order of transactions from any node *Ni* is preserved according to their physical timestamps. This eliminates the risk of malicious reordering between two or more transactions from the same node.  
2. **Batch-level ordering:** Within any batch of transactions output by the BFT committee, transactions are strictly ordered by timestamp.  
3. **Resistance to forged timestamps:** Nodes cannot freely manipulate timestamps due to the *TΔ* restriction window, preventing unrealistic backdating or future-dating of transactions.  

However, this design also introduces some trade-offs:  

- **Reduced throughput:** If the parameter *TΔ* is set poorly relative to varying network latencies, transactions may be unnecessarily aborted.  
- **Clock-related rejection:** Committee members can still exploit unsynchronized clocks by rejecting certain transactions. Nevertheless, members already possess the ability to reject transactions selectively.  
- **Risk of honest-node rejection:** Honest nodes with poorly synchronized clocks may reject valid transactions. This risk can be mitigated by enforcing stricter requirements for committee eligibility, such as proof of synchronized timekeeping.  

**Algorithm 1: Extra Verification Regarding Physical Timestamp**
```pseudo
Data: Input Transaction TX
Result: Boolean (verification passed or failed)

1  current_time ← Time.Now()
2  if |current_time - TX.Tp| > TΔ then
3      return false
4  var txn_history = static dictionary of lists
5  if txn_history[TX.from] == NULL then
6      txn_history[TX.from] = [TX]
7  else
8      if txn_history[TX.from][-1].Tp - TX.Tp > 0 then
9          return false
10     else
11         txn_history[TX.from].append(TX)
12         return true
```

### 3.5 Computation and Data Sharding, and Speculative Transaction Execution

Abey introduces sharding and speculative execution. Normal shards handle subsets of transactions, while a primary shard coordinates and finalizes. Shards do not overlap in membership. Each shard maintains state partitions. Transactions execute speculatively with lower and upper bounds, validated through two-phase commit across shards.

**Algorithm 2: Sharding and Speculative Transaction Processing**
```pseudo
On BecomeShard:
  Initialize state data: lastReaderTS = -1, lastWriterTS = -1

On transaction TX at shard Si:
  TX.lowerBound = 0
  TX.upperBound = ∞
  TX.state = RUNNING
  TX.before = []
  TX.after = []
  TX.ID = rand

On Read Address(addr):
  if host(addr) == Si:
    return local read
  else:
    broadcast readRemote(addr)
    wait for 2f+1 replies, else abort
  update TX.before and TX.lowerBound

On Write Address(addr):
  if host(addr) == Si:
    local write
  else:
    broadcast writeRemote(addr)
    wait for 2f+1 replies, else abort
  update TX.after and TX.lowerBound

On Finish:
  adjust bounds from before/after
  if lowerBound > upperBound: abort
  broadcast Precommit(ID, (lowerBound+upperBound)/2)
```

**Algorithm 3: Speculative Commit**
```pseudo
On receive Precommit(ID, cts):
  validate bounds, else abort
  update read/write timestamps
  broadcast Commit(ID, batchCounter)

On receive Commit(ID):
  mark TX committed
  update state metadata
  output log sorted by commit timestamp
```

This design ensures speculative execution remains serializable across shards.

---
## 4. Smart Contracts in Virtual Machines

### 4.1 Design Rationale

Smart contracts extend the blockchain from a passive ledger into a programmable platform for decentralized applications. Abey requires strict determinism: every node must produce identical outcomes for the same contract execution. Ethereum’s Yellow Paper [[12]](#ref-12) provided the early specification, although it is highly formal and difficult for developers to understand. KEVM [[13]](#ref-13) illustrates executable semantics, offering more clarity. By contrast, container-driven models like Hyperledger Fabric [[14]](#ref-14) encounter performance bottlenecks at high scale [[15]](#ref-15)–[[19]](#ref-19). Abey’s approach favors a virtual machine design tailored to scalability and portability.

### 4.2 Abey Virtual Machine (AVM)

The AVM is a deterministic, stack-based architecture modeled after the EVM [[20]](#ref-20), but adapted for Abey. It provides:

- **Determinism:** Contract execution yields identical results across all nodes.  
- **Compatibility:** Supports most EVM bytecode, easing migration.  
- **Native Cryptography:** Built-in elliptic curve cryptography.  
- **Hybrid Cloud Execution:** Offloads computationally intensive tasks with verifiable results.  
- **Scalability:** Deployable alongside Kubernetes [[15]](#ref-15) and OpenShift [[16]](#ref-16).  
- **Interfaces:** Works with Tendermint-style ABCI, DailyBFT consensus, permissioned EVMs, and RPC gateways.  

---

## 5. Blocks, State, and Transactions

Abey uses a block structure that records transactions, smart contract calls, committee membership data, and validator signatures. Each block is appended only after consensus is achieved. Key design elements include:

- **Block Contents:** Transactions, smart contracts, validator metadata, and cryptographic proofs.  
- **Parallel Execution:** Independent transactions can be executed in parallel, improving throughput.  
- **State Management:** The global state is a key-value store updated atomically by each transaction.  
- **Cross-Shard Verification:** Merkle proofs allow shards to verify state updates from other shards.  
- **Finality:** Once a quorum of committee members signs blocks, they are final and irreversible.  

Blocks propagate across the network, and validators check membership, digital signatures, and timestamp restrictions before committing them.

---
## 6. Incentive Design

Incentive structures encourage validator participation and fair behavior. Unlike proof-of-work, Abey redirects computational resources toward transaction scalability, state storage, and sharding.

### 6.1 Gas Fee and Sharding

- **Gas as Futures:** Gas fees are modeled as futures contracts to stabilize costs under heavy load.  
- **Gas Pools:** Rewards are distributed to validators via pooled gas fees collected from users.  
- **Liquidity Providers:** Entities may profit by anticipating stress in the network and supplying gas liquidity.  
- **Volunteer Computing Models:** Inspired by Gridcoin [[23]](#ref-23), Golem [[24]](#ref-24), BOINC [[25]](#ref-25), CERNVM [[26]](#ref-26), and LHC@Home [[27]](#ref-27), Abey integrates voluntary computing power into its reward model.  

---

## 7. Future Direction

Abey continues to evolve. The following areas represent priorities for future development:

- **Improved Timestamp Synchronization:** Ensuring physical and logical time consistency across shards and committees to prevent unfair transaction ordering.  
- **Refined Incentive Design:** Introducing dynamic gas pricing, enhanced futures markets, and cross-shard liquidity incentives.  
- **Replica-Based Sharding:** Exploring sharding models where replicas further improve reliability, fault tolerance, and scalability.  
- **Zero-Knowledge Proofs:** Integrating zk-SNARKs and zk-STARKs to strengthen privacy, scalability, and verifiability of off-chain computations.  
- **Hybrid Execution Environments:** Combining AVM with Linux containers to allow greater flexibility for decentralized applications and microservices.  
- **Enhanced Governance:** On-chain governance mechanisms allowing stakeholders to propose, deliberate, and vote on protocol upgrades.  
- **Cross-Chain Interoperability:** Bridges and standards enabling Abey to interact with other major blockchain ecosystems.  

---

## 8. Conclusions

Abey 3.0 presents a robust, scalable proof-of-stake protocol designed for the future of decentralized applications. Its innovations include:

- **Consensus:** Rotating PoS committees with fair elections, instant finality, and resilience to adversarial behavior.  
- **Virtual Machine:** The Abey Virtual Machine (AVM), offering determinism, compatibility with EVM, and hybrid cloud execution.  
- **Sharding:** Computation and data sharding with speculative execution to maximize throughput and maintain serializability.  
- **Incentives:** A redesigned gas market model using futures and volunteer computing frameworks.  
- **Scalability:** Parallel execution of independent transactions and verifiable cross-shard communication.  
- **Extensibility:** Integration paths for zero-knowledge proofs, enhanced governance, and cross-chain communication.  

These features together establish Abey as a next-generation public blockchain infrastructure, combining efficiency, fairness, and extensibility to meet the needs of modern decentralized ecosystems.

---

## References

1. <a id="ref-1"></a> S. Nakamoto. *Bitcoin: A Peer-to-Peer Electronic Cash System*. [http://bitcoin.org/bitcoin.pdf](http://bitcoin.org/bitcoin.pdf), 2008.  
2. <a id="ref-2"></a> V. Buterin. *Ethereum White Paper*. [https://github.com/ethereum/wiki/wiki/White-Paper](https://github.com/ethereum/wiki/wiki/White-Paper), 2014.  
3. <a id="ref-3"></a> M. Castro, B. Liskov, et al. *Practical Byzantine Fault Tolerance*. In *OSDI*, vol. 99, pp. 173–186, 1999.  
4. <a id="ref-4"></a> Id. at 173–186.  
5. <a id="ref-5"></a> E. Androulaki, A. Barger, V. Bortnikov, et al. *Hyperledger Fabric: A Distributed Operating System for Permissioned Blockchains*. [https://arxiv.org/pdf/1801.10228v1.pdf](https://arxiv.org/pdf/1801.10228v1.pdf), 2018.  
6. <a id="ref-6"></a> R. Pass, E. Shi. *Hybrid Consensus: Efficient Consensus in the Permissionless Model*. In *LIPIcs - Leibniz International Proceedings in Informatics*, vol. 91. Schloss Dagstuhl–Leibniz-Zentrum für Informatik, 2017.  
7. <a id="ref-7"></a> R. Canetti. *Universally Composable Security: A New Paradigm for Cryptographic Protocols*. In *Proceedings of the 42nd IEEE Symposium on Foundations of Computer Science*, pp. 136–145, IEEE, 2001.  
8. <a id="ref-8"></a> R. Pass, E. Shi. *Hybrid Consensus: Efficient Consensus in the Permissionless Model*. In *LIPIcs - Leibniz International Proceedings in Informatics*, vol. 91. Schloss Dagstuhl–Leibniz-Zentrum für Informatik, 2017.  
9. <a id="ref-9"></a> R. Pass, E. Shi. *Thunderella: Blockchains with Optimistic Instant Confirmation*. [https://eprint.iacr.org/2017/913.pdf](https://eprint.iacr.org/2017/913.pdf), 2017.  
10. <a id="ref-10"></a> X. Yu, A. Pavlo, D. Sanchez, S. Devadas. *TicToc: Time Traveling Optimistic Concurrency Control*. In *SIGMOD*, pp. 1629–1642, ACM, 2016.  
11. <a id="ref-11"></a> H. A. Mahmoud, V. Arora, F. Nawab, D. Agrawal, A. El Abbadi. *MaaT: Effective and Scalable Coordination of Distributed Transactions in the Cloud*. *VLDB Endowment*, 7(5):329–340, 2014.  
12. <a id="ref-12"></a> G. Wood. *Ethereum: A Secure Decentralized Generalized Transaction Ledger (Yellow Paper)*. [https://ethereum.github.io/yellowpaper/paper.pdf](https://ethereum.github.io/yellowpaper/paper.pdf), 2018.  
13. <a id="ref-13"></a> E. Hildenbrandt, M. Saxena, X. Zhu, et al. *KEVM: A Complete Semantics of the Ethereum Virtual Machine*. [https://www.ideals.illinois.edu/handle/2142/97207](https://www.ideals.illinois.edu/handle/2142/97207), 2017.  
14. <a id="ref-14"></a> E. Androulaki, A. Barger, V. Bortnikov, et al. *Hyperledger Fabric: A Distributed Operating System for Permissioned Blockchains*. [https://arxiv.org/pdf/1801.10228v1.pdf](https://arxiv.org/pdf/1801.10228v1.pdf), 2018.  
15. <a id="ref-15"></a> Kubernetes Documentation. *Building Large Clusters*. [https://kubernetes.io/docs/admin/cluster-large/](https://kubernetes.io/docs/admin/cluster-large/).  
16. <a id="ref-16"></a> Red Hat. *OpenShift Container Platform: Cluster Limits*. [https://access.redhat.com/documentation/en-us/openshift_container_platform/3.9/html/scaling_and_performance_guide/](https://access.redhat.com/documentation/en-us/openshift_container_platform/3.9/html/scaling_and_performance_guide/).  
17. <a id="ref-17"></a> Red Hat Storage. *Container-Native Storage for the OpenShift Masses*. [https://redhatstorage.redhat.com/2017/10/05/containernative-storage-for-the-openshift-masses/](https://redhatstorage.redhat.com/2017/10/05/containernative-storage-for-the-openshift-masses/).  
18. <a id="ref-18"></a> Kubernetes GitHub Issue. *Increase Maximum Pods per Node: kubernetes/kubernetes#23349*. [https://github.com/kubernetes/kubernetes/issues/23349](https://github.com/kubernetes/kubernetes/issues/23349).  
19. <a id="ref-19"></a> OpenShift Blog. *Deploying 2048 OpenShift Nodes on the CNCF Cluster*. [https://blog.openshift.com/deploying-2048-openshift-nodes-cncf-cluster/](https://blog.openshift.com/deploying-2048-openshift-nodes-cncf-cluster/). See also: *Kubernetes Scalability Goals*. [https://github.com/kubernetes/community/blob/master/sig-scalability/goals.md](https://github.com/kubernetes/community/blob/master/sig-scalability/goals.md). 
20. <a id="ref-20"></a> G. Wood. *Ethereum: A Secure Decentralized Generalized Transaction Ledger (Yellow Paper)*. [https://ethereum.github.io/yellowpaper/paper.pdf](https://ethereum.github.io/yellowpaper/paper.pdf), 2018.  
21. <a id="ref-21"></a> T. Schmidt. *Modeling Energy Markets with Extreme Spikes*. In: A. Sarychev, A. Shiryaev, M. Guerra, M. R. Grossinho (eds). *Mathematical Control Theory and Finance*, pp. 359–375. Springer, Berlin, Heidelberg, 2008.  
22. <a id="ref-22"></a> W. Delbaen, F. Schachermayer. *A General Version of the Fundamental Theorem of Asset Pricing*. *Mathematische Annalen*, 300(1):463–520, 1994.  
23. <a id="ref-23"></a> Gridcoin. *Gridcoin Whitepaper: The Computation Power of a Blockchain Driving Science and Data Analysis*. [https://www.gridcoin.us/assets/img/whitepaper.pdf](https://www.gridcoin.us/assets/img/whitepaper.pdf).  
24. <a id="ref-24"></a> Golem Project Team. *The Golem Project Whitepaper*. [https://golem.network/doc/Golemwhitepaper.pdf](https://golem.network/doc/Golemwhitepaper.pdf), 2016.  
25. <a id="ref-25"></a> D. P. Anderson. *BOINC: A System for Public-Resource Computing and Storage*. [https://boinc.berkeley.edu/grid_paper_04.pdf](https://boinc.berkeley.edu/grid_paper_04.pdf), 2004.  
26. <a id="ref-26"></a> J. Blomer, L. Franco, A. Harutyunian, P. Mato, Y. Yao, C. Aguado Sanchez, P. Buncic. *CERNVM – A Virtual Software Appliance for LHC Applications*. [http://iopscience.iop.org/article/10.1088/1742-6596/219/4/042003/pdf](http://iopscience.iop.org/article/10.1088/1742-6596/219/4/042003/pdf), 2017.  
27. <a id="ref-27"></a> D. Lombraña González, et al. *LHC@Home: A Volunteer Computing System for Massive Numerical Simulations of Beam Dynamics and High-Energy Physics Events*. [http://inspirehep.net/record/1125350/](http://inspirehep.net/record/1125350/).  
