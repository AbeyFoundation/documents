# Abey Blockchain Whitepaper

## Table of Contents
- [Abstract](#abstract)
- [1. Introduction](#1-introduction)
- [2. Background](#2-background)
  - [2.1 Related Works](#21-related-works)
- [3. Consensus](#3-consensus)
  - [3.1 Design Overview](#31-design-overview)
  - [3.2 Recap of DPoS Consensus Protocol](#32-recap-of-dpos-consensus-protocol)
    - [3.2.1 Daily Off-Chain Consensus Protocol](#321-daily-off-chain-consensus-protocol)
    - [3.2.2 Mempool Subprotocol](#322-mempool-subprotocol)
    - [3.2.3 Main Consensus Protocol](#323-main-consensus-protocol)
  - [3.3 Variant Day Length and Committee Election](#33-variant-day-length-and-committee-election)
  - [3.4 Application-Specific Design](#34-application-specific-design)
    - [3.4.1 Physical Timing Restriction](#341-physical-timing-restriction)
  - [3.5 Computation and Data Sharding, and Speculative Transaction Execution](#35-computation-and-data-sharding-and-speculative-transaction-execution)
- [4. Smart Contracts in Virtual Machines](#4-smart-contracts-in-virtual-machines)
  - [4.1 Design Rationale](#41-design-rationale)
  - [4.2 Abey Chain Virtual Machine (AVM)](#42-abey-chain-virtual-machine-avm)
- [5. Blocks, State, and Transactions](#5-blocks-state-and-transactions)
- [6. Incentive Design](#6-incentive-design)
  - [6.1 Gas Fee and Sharding](#61-gas-fee-and-sharding)
- [7. Future Directions](#7-future-directions)
- [8. Conclusions](#8-conclusions)
- [References](#references)

---

## Abstract

In this paper, we present the initial design of Abey Blockchain (3.0+) and its technical details. Our consensus design guarantees consistency, liveness, transaction finality, and security. We discuss optimizations such as the frequency of rotating committee members and physical timestamp restrictions. The primary focus of Abey Blockchain is to advance these concepts and build a blockchain uniquely designed for the Abey community. We also utilize: (i) data sharding and speculative transactions, (ii) evaluation of smart contracts in a hybrid cloud infrastructure, and (iii) volunteer computing protocols for a new compensation infrastructure.

---

## 1. Introduction

With the surging popularity of cryptocurrencies, blockchain technology has caught the attention of both industry and academia. Blockchain can be viewed as a shared computing environment involving peers that can freely join and leave, governed by a consensus protocol. Its decentralized nature, along with transaction transparency, autonomy, and immutability, forms the foundation of cryptocurrencies.

However, earlier cryptocurrencies such as Bitcoin [[1]](#references) and Ethereum [[2]](#references) are widely recognized as unscalable in terms of transaction rate and are economically inefficient due to high energy and computation demands.

With growing demand for public blockchain applications, a viable protocol enabling higher transaction rates is essential. For example, a public blockchain that hosts peer-to-peer gaming applications and smart contracts for Initial Coin Offerings (ICOs) would face severe delays in transaction confirmation without improved scalability.

Alternative models such as Delegated Proof-of-Stake (DPoS) and Hybrid Consensus have emerged. Abey Blockchain 2.0 adopts hybrid consensus, combining a modified Practical Byzantine Fault Tolerance (PBFT) [[3]](#references) with Proof-of-Work (PoW). Abey Blockchain 3.0 phases out energy-intensive PoW in favor of the more efficient Proof-of-Stake (PoS). PoS with validators enables high throughput, is energy efficient, and remains secure as long as no more than one-third of validators are malicious at any time.

In this paper, we propose Abey Blockchain 3.0, a PoS protocol where validators achieve consensus using a modified PBFT [[4]](#references). This system incentivizes validators, ensures fair committee rotation, provides instant finality with high throughput, and supports a compensation infrastructure for non-uniform resources. The protocol tolerates corruption of up to one-third of nodes.

---

## 2. Background

The strength of this proposal lies in recognizing the theorems of DPoS. By using DailyBFT as committee members, Abey Blockchain supports rotating committees, enhancing fairness among consensus validators.

### 2.1 Related Works

In PoS systems, the native token represents both value and voting power, unlike PoW systems where tokens only store value. PoS achieves Sybil resistance via Byzantine Fault Tolerance (BFT) while consuming significantly less energy. Participation in PoS depends on token staking, reducing computational costs compared to PoW. Validators are chosen proportionally to their stake, offering notable benefits and trade-offs distinct from PoW systems.

---

## 3. Consensus

Our consensus design is a PoS model with modifications to support application-specific requirements. We assume readers are familiar with PoS fundamentals.

### 3.1 Design Overview

The protocol builds on Hybrid Consensus [[5]](#references). Our adversary model allows mild adaptive corruption of nodes, with delayed effects [[6]](#references). Future versions will formally define modifications under the Universal Composability model [[7]](#references).

*Note: All pseudocode presented here is simplified and not optimized for engineering.*

### 3.2 Recap of DPoS Consensus Protocol

The Abey Blockchain 3.0 PoS consensus protocol includes several subprotocols:

#### 3.2.1 Daily Off-Chain Consensus Protocol

In DailyBFT, committee members run an off-chain BFT instance to decide a daily log, while non-members validate signatures. This provides security for non-members and late-joining nodes. Committee members output signed daily log hashes, consumed by the PoS consensus protocol. These signatures satisfy completeness and unforgeability.

- **BFT Member Behavior**: A BFT virtual node (*BFTpk*) processes transactions (TXs). If a stop signal is signed by at least one-third of committee keys, logging ceases. Gossip checks ensure consensus.
- **Non-BFT Member Behavior**: Transactions are added to history and signed by one-third of committee keys.

The signing algorithm tags inner BFT messages with prefix "0" and DailyBFT messages with prefix "1" to prevent namespace collisions.

#### 3.2.2 Mempool Subprotocol

The mempool initializes transactions with 0 and tracks them in a set. On receiving a proposal, transactions are added to blocks and disseminated via gossip. Confirmed transactions are purged, and query methods return verified transactions.

#### 3.2.3 Main Consensus Protocol

A new node inherits message history and interacts with mempools, preprocessing, off-chain consensus, and on-chain validation.

### 3.3 Variant Day Length and Committee Election

Committees rotate after fixed periods (logical chain-based clocks) [[8]](#references). Miners of the latest *csize* blocks form new committees. Our design reduces switching frequency to minimize overhead, but enforces switches to maintain fairness. Misbehavior triggers forced switches, informed by Thunderella [[9]](#references).

### 3.4 Application-Specific Design

Consensus adapts to specific application requirements without compromising consistency, liveness, or security.

#### 3.4.1 Physical Timing Restriction

Committee members may reorder transactions within small time windows, risking fairness in high-throughput applications like exchanges. To mitigate this, Abey Blockchain requires sticky timestamps (*TΔ*). Transactions include signed physical timestamps (*Tp*), verified by validators.

**Algorithm 1: Extra Verification Regarding Physical Timestamp**
```pseudo
Data: Input Transaction TX
Result: Boolean (verification passed or failed)

1  current_time ← Time.Now()
2  if |current_time - TX.Tp| > TΔ then
3      return false   // Reject if time skew is too large
4  var txn_history = new static dictionary of lists
5  if txn_history[TX.from] == NULL then
6      txn_history[TX.from] = [TX]
7  else
8      if txn_history[TX.from][-1].Tp - TX.Tp > 0 then
9          return false   // Enforce order from same node
10     else
11         txn_history[TX.from].append(TX)
12         return true
```

### 3.5 Computation and Data Sharding, and Speculative Transaction Execution

**Algorithm 2: Sharding and Speculative Transaction Processing**
```pseudo
On BecomeShard:
  Initialize all the state data sectors: lastReaderTS = -1, lastWriterTS = -1, readers = [], writers = []

On transaction TX on shard Si:
  On Initialization:
    TX.lowerBound = 0
    TX.upperBound = ∞
    TX.state = RUNNING
    TX.before = []
    TX.after = []
    TX.ID = rand

  On Read Address(addr):
    if host(addr) == Si then
        Send readRemote(addr) to itself
    else
        Broadcast readRemote(addr, TX.id) to host(addr)
        Async wait for 2f+1 replies within timeout To
        Abort TX on timeout
    Let val, wts, IDs = majority reply
    TX.before.append(IDs)
    TX.lowerBound = max(TX.lowerBound, wts)
    return val

  On Write Address(addr):
    if host(addr) == Si then
        Send writeRemote(addr) to itself
    else
        Broadcast writeRemote(addr, TX.id) to host(addr)
        Async wait for 2f+1 replies within timeout To
        Abort TX on timeout
    Let rts, IDs = majority reply
    TX.after.append(IDs)
    TX.lowerBound = max(TX.lowerBound, rts)
    return

  On Finish Execution:
    for every TX’ in TX.before:
        TX.lowerBound = max(TX.lowerBound, TX’.upperBound)
    for every TX’ in TX.after:
        TX.upperBound = min(TX.upperBound, TX’.lowerBound)
    if TX.lowerBound > TX.upperBound:
        Abort TX
    Broadcast Precommit(TX.ID, (TX.lowerBound+TX.upperBound)/2)
```

**Algorithm 3: Sharding and Speculative Transaction Processing (cont.)**
```pseudo
On receive Precommit(ID, cts):
  Look up TX by ID
  if Found and cts not in [TX.lowerBound, TX.upperBound]:
      Broadcast Abort(ID) to sender’s shard
  TX.lowerBound = TX.upperBound = cts
  For each data sector DS[addr] TX reads:
      DS[addr].rts = max(DS[addr].rts, cts)
  For each data sector DS[addr] TX writes:
      DS[addr].wts = max(DS[addr].wts, cts)
  Broadcast Commit(ID, batchCounter)

On receive 2f+1 Commit(ID, batchCounter) from each remote shard accessed:
  TX.lowerBound = TX.upperBound = cts
  For each DS[addr] TX reads:
      DS[addr].rts = max(DS[addr].rts, cts)
  For each DS[addr] TX writes:
      DS[addr].wts = max(DS[addr].wts, cts)
  Mark TX committed
  TX.metadata = [ShardID, batchCounter]

On output log:
  Sort TXs by cts, break ties by timestamp
```

---

## 4. Smart Contracts in Virtual Machines

### 4.1 Design Rationale

Smart contracts extend the blockchain beyond a simple transaction ledger, enabling decentralized applications. To achieve this, Abey Blockchain employs a hybrid execution model that leverages both on-chain determinism and off-chain cloud scalability.

### 4.2 Abey Chain Virtual Machine (AVM)

The AVM provides a deterministic execution environment for smart contracts. It ensures compatibility with Ethereum Virtual Machine (EVM) standards while offering extensions to support high-performance workloads in hybrid cloud infrastructures. It integrates features for resource monitoring, dynamic scaling, and interaction with external services.

---

## 5. Blocks, State, and Transactions

Abey Blockchain organizes data in blocks, with each block containing transactions and state updates. Blocks are validated by committees using the PoS protocol. The state is maintained as a global key-value store, updated atomically with each block. Transactions include metadata for ordering, timestamps, and cryptographic proofs.

---

## 6. Incentive Design

Abey Blockchain incentivizes participants through transaction fees, block rewards, and a volunteer computing reward infrastructure. This ensures fairness and sustainability while encouraging participation from diverse stakeholders.

### 6.1 Gas Fee and Sharding

Gas fees are designed to reflect computational costs and storage use. Sharding reduces transaction costs by distributing workloads. Fees collected are shared among validators and contributors, ensuring balanced incentives.

---

## 7. Future Directions

Future improvements include advanced sharding algorithms, enhanced interoperability with other blockchains, zero-knowledge proof integration for privacy, and broader adoption of volunteer computing for resource sharing.

---

## 8. Conclusions

Abey Blockchain 3.0 introduces a scalable, energy-efficient PoS protocol with strong guarantees of finality, security, and fairness. By integrating committee-based consensus, sharding, hybrid smart contracts, and innovative incentive structures, it offers a powerful platform for decentralized applications.

---

## References

1. S. Nakamoto. [Bitcoin: A Peer-to-Peer Electronic Cash System (2008)](http://bitcoin.org/bitcoin.pdf)
2. V. Buterin. [Ethereum White Paper (2014)](https://github.com/ethereum/wiki/wiki/White-Paper)
3. M. Castro, B. Liskov. [Practical Byzantine Fault Tolerance (1999)](https://pmg.csail.mit.edu/papers/osdi99.pdf)
4. R. Pass, E. Shi. [Hybrid Consensus (2017)](https://drops.dagstuhl.de/entities/document/10.4230/LIPIcs.DISC.2017.39)
5. E. Androulaki et al. [Hyperledger Fabric (2018)](https://arxiv.org/pdf/1801.10228v1.pdf)
6. R. Pass, E. Shi. [Hybrid Consensus in Permissionless Model (2017)](https://drops.dagstuhl.de/opus/volltexte/2017/7963/pdf/LIPIcs-DISC-2017-39.pdf)
7. R. Canetti. [Universally Composable Security (2001)](https://eprint.iacr.org/2000/067.pdf)
8. X. Yu et al. [TicToc: Time Traveling Optimistic Concurrency Control (2016)](https://dl.acm.org/doi/10.1145/2882903.2882931)
9. R. Pass, E. Shi. [Thunderella: Blockchains with Optimistic Instant Confirmation (2017)](https://eprint.iacr.org/2017/913.pdf)
10. H. Mahmoud et al. [Maat: Effective Coordination of Distributed Transactions (2014)](https://dl.acm.org/doi/10.14778/2732269.2732276)
11. G. Wood. [Ethereum Yellow Paper (2018)](https://ethereum.github.io/yellowpaper/paper.pdf)
12. E. Hildenbrandt et al. [KEVM: Semantics of the EVM (2017)](https://www.ideals.illinois.edu/items/97207)
13. Gridcoin. [Gridcoin Whitepaper (2017)](https://www.gridcoin.us/assets/img/whitepaper.pdf)
14. Golem Team. [Golem Project Whitepaper (2016)](https://golem.network/doc/Golemwhitepaper.pdf)
15. D. P. Anderson. [BOINC: Public Resource Computing (2004)](https://boinc.berkeley.edu/grid_paper_04.pdf)
16. J. Blomer et al. [CERNVM Appliance for LHC (2017)](http://iopscience.iop.org/article/10.1088/1742-6596/219/4/042003/pdf)
17. D. Lombraña González et al. [LHC@Home Volunteer Computing (2012)](http://inspirehep.net/record/1125350/)
18. Kubernetes Documentation. [Building Large Clusters](https://kubernetes.io/docs/admin/cluster-large/)
19. Red Hat. [OpenShift Container Platform Cluster Limits](https://access.redhat.com/documentation/enus/openshift_container_platform/3.9/html/scaling_and_performance_guide/)
20. Red Hat. [Container-Native Storage for OpenShift](https://redhatstorage.redhat.com/2017/10/05/containernative-storage-for-the-openshift-masses/)
21. Kubernetes. [Increase Maximum Pods per Node: Issue #23349](https://github.com/kubernetes/kubernetes/issues/23349)
22. OpenShift Blog. [Deploying 2048 OpenShift Nodes on CNCF Cluster](https://blog.openshift.com/deploying-2048-openshift-nodes-cncf-cluster/)
23. T. Schmidt. [Modeling Energy Markets with Extreme Spikes (2008)](https://link.springer.com/chapter/10.1007/978-3-540-78999-6_16)
24. W. Delbaen, F. Schachermayer. [Fundamental Theorem of Asset Pricing (1994)](https://doi.org/10.1007/BF01446532)
25. E. Androulaki et al. [Hyperledger Fabric: Distributed OS (2018)](https://arxiv.org/pdf/1801.10228v1.pdf)
26. G. Wood. [Ethereum Yellow Paper (2018)](https://ethereum.github.io/yellowpaper/paper.pdf)
27. E. Hildenbrandt et al. [KEVM Extended

