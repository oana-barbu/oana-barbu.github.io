---
layout: default
title: Network Latency as the Silent Centralizer
---

## Why Sub-Second Block Times are Incompatible with Decentralization

Blockchains rely on networks of nodes exchanging messages to agree on state and make progress. These messages travel over the Internet, the global system of “pipes” that carry messages from one place to another. Some paths are short and wide (low latency, high bandwidth); others are long and narrow (prone to congestion and slow). In best-case conditions, a message may take 5-15 ms to cross a country, 35-50 ms to cross the Atlantic, 90-125 ms from Europe to East Asia, or 150-160 ms from South America to Australia[^1].

These figures come from datacenter-to-datacenter traffic on private backbones operated by large cloud providers, where traffic gets dedicated, high-speed pipes. On the public Internet, one-way delays are typically higher due to indirect routes, multiple hops, congestion, and packet loss. If you're deep in the Amazon rainforest pinging a friend on a Lord of the Rings pilgrimage in New Zealand, latency can be far above these lower bounds.

Most everyday applications hide this reality by relying on Content Delivery Networks (CDNs). CDNs cache content on edge servers close to users to serve requests faster. When you load google.com from Asia, you’re not talking to a server in the US, but to a nearby point of presence in Japan, Singapore, Hong Kong, or wherever is closest. Blockchains cannot use CDNs in this way. Consensus requires fresh, real-time data to stay in sync. As a result, physical distance is unavoidable. If one blockchain node is in Sydney and another in São Paulo, any message between them will incur at least 150-160 ms of one-way delay in ideal conditions, and often more on public-Internet paths, regardless of software optimizations.

Moreover, reaching consensus (the process by which nodes agree on the canonical chain state) does not happen in a single message exchange. Most consensus protocols, especially Byzantine Fault Tolerant (BFT) designs, are multi-phase: a leader proposes a block, nodes vote on it, and a final phase commits the result. These steps are strictly sequential: votes require seeing the proposal, and finalization requires seeing both the proposal and the votes. As a result, the absolute minimum time to reach consensus is roughly the number of phases multiplied by the effective network delay, which we can approximate as **3 × one-way delay** if propagation dominates.

This sets a lower bound on block time: network delay times the number of sequential phases. In practice, block times are higher due to transaction execution, vote aggregation, timeout margins, and other protocol overheads. And that’s only the beginning of the problem.

First, network delay is not uniform across participants. Even if node A can reach node B in 10 ms one way, if most nodes are 50 ms away, each consensus phase must wait 50 ms to ensure broad participation. Protocols therefore size their timeouts around the network’s effective diameter (the maximum one-way delay between any two nodes), not the fastest links. Additionally, latency is not stable. The time to traverse a pipe depends on the pipe itself and what's sharing it right now. If the pipe is empty, you're in luck; if it's peak hour and everyone's streaming their favorite shows or downloading updates, the pipe gets crowded, queues build up, packets are dropped, and retransmissions add additional delay (think doubling or worse the message latency). To preserve safety and liveness under these real-world conditions, consensus protocols must budget for worst-case delay, jitter, and loss. The unavoidable consequence is longer block times than the theoretical minimum.

Second, how messages propagate across blockchain nodes directly affects latency. There are three common approaches:

- **Direct messaging:** The fastest way to send a message is to send it directly to everyone who needs it. This is manageable in a small network with a few tens of nodes, but it scales horribly. In a network with one million nodes, each sender would need to transmit on the order of a million messages per event. Infeasible.
- **Gossiping:** Each node forwards messages only to a small set of neighbors. This scales well with respect to per-node bandwidth requirements but incurs higher latency. In a million-node network, gossip typically requires 20-25 hops to reach most participants[^2]. With an average hop delay of 50-100 ms, one-way propagation alone takes 1-2.5 seconds. Since multi-phase consensus requires several such propagations, dissemination time alone can extend to multiple seconds.
- **The hybrid approach:** Nodes send messages to a predetermined subset of peers (determined by stake or other criteria), requiring fewer messages per sender than direct messaging, while providing more predictable delivery than random gossip. In Solana Alpenglow, for example, the leader sends to a stake-weighted set of relayers, who then rebroadcast to everyone else in a second hop. Assuming the same 50-100ms per hop, messages reach the network in 100-200 ms, far faster than pure gossip. In reality, though, in a low-hop setup, if a node is significantly farther, the tail latency spikes. If that node happens to be a relayer, then the latency of all nodes connected to it increases.

Solana’s propagation design is faster than pure gossip and scales better than full-mesh direct messaging, but the scalability trade-off remains. In a hypothetical network with 1 million validators, reaching the entire network would still require nodes to send each message to thousands of peers, imposing significant bandwidth demands on the sending nodes.

Solana can accommodate this because blocks are split into small (~1.2 KB) shreds and streamed incrementally. With shreds, fanning out to 1000 peers is feasible: for example, with a 10 ms streaming window per shred, the required upstream bandwidth is on the order of 1 Gbps, which is manageable. The same fanout would be far more demanding in systems like Ethereum, where blocks are propagated as whole objects and can be hundreds of KB.

Even so, a 10 ms per-shred window is already large. With an average transaction size of 0.5 KB, emitting one shred every 10 ms (100 shreds/sec) limits throughput to roughly 240 TPS. Solana achieves thousands of TPS by shrinking the effective per-shred window, thereby increasing bandwidth requirements.

By now, the pattern should be clear: blockchains with low latency tend to have either:
- few nodes (making direct communication feasible)
- nodes clustered geographically (minimizing propagation delay)

Or both! The inverse also holds: as the number of nodes grows and their geographic spread increases, latency rises. But a large number of nodes and a wide geographic distribution are exactly what we usually mean by decentralization. This leads to the central conjecture of this piece: **greater decentralization necessarily implies higher latency**.

![Decentralization Capacity vs. Minimum Block Time](decentralization-vs-block-time.svg)

Consider a few concrete examples.

#### Hyperliquid

Hyperliquid advertises a block time of 0.07 seconds, with median latency around 0.1 seconds. Supporting a three-phase BFT-style protocol within that budget implies an average one-way network delay below 25 ms, even before accounting for off-wire processing. In practice, this requires validators to co-locate in a single region. Indeed, the official setup documentation strongly encourages running nodes in Tokyo[^3].

Hyperliquid also employs a jailing mechanism: validators can temporarily exclude peers that respond too slowly to consensus messages, ensuring that only low-latency nodes actively participate. The validator set is small (21 active validators) because Hyperliquid's HyperBFT relies on direct communication among validators.

The result is ~100 ms block times, achieved at the cost of extreme centralization.

#### Solana

Solana's new consensus protocol, Alpenglow[^4], targets 150 ms median block finality, with best-case outcomes as low as 100 ms. It runs two concurrent paths:
- Happy path: Proposal + one voting round, finalizing if ≥80% of stake responds (2 phases).
- Fallback path: proposal + two voting rounds, finalizing if only ≥60% of stake responds (3 phases).

Finality occurs as soon as the faster path completes. Simulations using the current mainnet stake distribution, with the leader located in Zurich, show that ~65% of the stake lies within a 50 ms one-way network delay of Zurich. This matches observed reality: roughly 60-70% of Solana's delegated stake is hosted in Europe[^5], with strong low-latency connectivity to central Europe. Achieving sub-150 ms finality, therefore, depends on significant geographic stake clustering.

Solana already has high operational requirements for validators and a relatively small validator set (~800). Substantially increasing the validator count would further stress the networking capacity of the nodes responsible for block dissemination.

#### Ethereum

Ethereum's block time is 12 seconds. Each slot involves three sequential communication phases, and the network includes over one million active validators, with message propagation handled via GossipSub. In a network of this size, reaching all nodes typically requires 20-25 hops. To complete each phase within the 12-second slot, the average one-way delay per hop is around 160-200 ms: this is well within what the public Internet supports and imposes minimal constraints on where validators can be located. In practice, most hops are much shorter (10-50 ms within regions), with only a minority being long-haul (100-150 ms or more across continents). A 200 ms one-way delay is sufficient to span most of the globe, and locality-aware peering further biases propagation toward closer neighbors. This leaves Ethereum room to reduce block times somewhat (e.g., proposals for 6-second slots in future upgrades) without significantly compromising decentralization.

Sub-second block times, however, are a different story. With Ethereum’s 3-phase consensus, gossip would need to reach one million nodes in ~330ms per phase, assuming most time is spent on communication. That implies an average one-way delay per hop of 10-15 ms across the entire network, which requires strong geographic clustering and conflicts with Ethereum’s goal of global decentralization.

#### Bitcoin

With a 10-minute block interval, there is plenty of time for messages to propagate across the network, even if you're on top of Everest (assuming reception). Bitcoin’s Nakamoto consensus tolerates high latency, jitter, and packet loss, so there is no pressure to cluster nodes geographically. Nodes can be run from home connections or constrained environments, without risking consensus failure due to network conditions. The requirements to run a full node are correspondingly low: a basic machine, modest storage, and a standard internet connection are sufficient. There is no staking, no strict uptime requirement, and no need for professional networking setups. The result is strong global decentralization, at the cost of throughput and latency.


### Other considerations

#### Direct messaging and DoS risk

Direct messaging requires nodes to know how to reach specific peers, which in practice means exchanging IP addresses. Merely knowing the IPs of random gossip peers is not inherently dangerous. The risk arises when a stable validator identity, its protocol role (e.g., proposer or high-weight voter), and its IP address are tightly bound together. Once an attacker can map a validator to its IP during a critical moment, denial-of-service attacks become much easier. Mitigating such attacks reliably requires professional infrastructure (sentry nodes, traffic filtering, and overprovisioned bandwidth), which favors well-resourced operators. Validators in Solana and Hyperliquid are expected to operate this manner, whereas Ethereum and Bitcoin nodes are often consumer-run and do not require such measures. Consequently, protocols that rely on direct messaging implicitly raise the bar for participation and introduce a centralizing pressure.

#### Failure to participate and network assumptions

Ethereum penalizes validators for inactivity through slashing and inactivity leaks[^6], whereas Solana and Hyperliquid do not: validators may miss rewards or be temporarily excluded, but their stake is not burned.

Ethereum’s inactivity leak is an automatic liveness mechanism that progressively reduces the stake of non-participating validators, allowing online validators to regain a supermajority and finalize, even under widespread or correlated outages. These harsher penalties force Ethereum’s designers to be more conservative about network assumptions. Real-world networks are unreliable: latency fluctuates, packets are dropped, and jitter is common, especially for non-professional setups. Even if the theoretical minimum one-way delay is 10 ms, practical designs must add safety margins. Longer block times provide this margin, enabling broader global participation from nodes with weaker connectivity and fewer operational guarantees.

### Why global decentralization matters

You might think ultra-fast blockchains can ignore geography, but real-world events tell a different story. For example, in April 2025, Portugal, Spain, and parts of France experienced a 12-hour power outage. Had this happened in Japan, Hyperliquid would likely have been offline during that time.

The core premise of blockchains is continuous availability without trusted infrastructure. Geographic concentration undermines that promise: highly optimized, low-latency networks are fast, but fragile.

### TL;DR

Low block times require low latency between blockchain participants, which in practice demands geographical centralization. Internet physics (long-distance pipes, jitter, packet loss) and consensus mechanics (multi-phase rounds, propagation hops) create a hard trade-off:
- Fast block times (sub-second) require tightly clustered nodes to keep one-way delays under 50 ms.
- Global decentralization (millions of validators spread worldwide) forces higher latency tolerance, longer block times, and generous margins for network variability.

Speed comes with a decentralization tax. You can't have both ultra-low latency and a truly worldwide, permissionless network.

---

[^1]: [Azure Network Round-Trip Latency Statistics](https://learn.microsoft.com/en-us/azure/networking/azure-network-latency)

[^2]: [Demers et al. "Epidemic Algorithms for Replicated Database Maintenance" (1987)](https://dl.acm.org/doi/10.1145/41840.41841)

[^3]: [Hyperliquid Node Documentation](https://github.com/hyperliquid-dex/node)

[^4]: [Anza: Alpenglow - A New Consensus for Solana](https://www.anza.xyz/blog/alpenglow-a-new-consensus-for-solana)

[^5]: [Helius: Measuring Solana's Decentralization](https://www.helius.dev/blog/solana-decentralization-facts-and-figures)

[^6]: [Ethereum.org: Proof-of-Stake Rewards and Penalties](https://ethereum.org/developers/docs/consensus-mechanisms/pos/rewards-and-penalties/)
