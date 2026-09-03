# Narrow-Space Coordination: Bottleneck Analysis & Theoretical Solutions
**SIH 2026 PS 26123 — Decentralized Multi-AMR Fleet Architecture**[cite: 1]

---

## 1. Problem Associated with the Baseline Algorithm

The baseline architecture handles narrow aisles, single-lane intersections, and lifts using a **Ricart-Agrawala-style distributed mutual exclusion protocol** ordered by Lamport logical timestamps[cite: 1]. 

While mathematically starvation-free and deadlock-free in fully connected static networks, running this protocol fleet-wide across mobile warehouse robots introduces critical bottlenecks[cite: 1]:

* **$2(N-1)$ Global Message Scaling:** To enter a single shared corridor, an AMR must broadcast a `REQUEST` to all $N-1$ peers in the fleet and collect an explicit `GRANT` response from every single one before it is permitted to cross the entrance boundary[cite: 1]. In a fleet of 10 AMRs, entering one aisle generates 18 dedicated wireless messages; for 20 AMRs, it demands 38 messages per entry.
* **Corridor Entrance Idle Latency:** Robots must throttle down or come to a full standstill at aisle entry thresholds while waiting for synchronous round-trip message delivery across the entire warehouse[cite: 1]. This creates physical queuing delays, increasing overall task makespan[cite: 1].
* **Sensitivity to Distant Packet Loss:** If an AMR operating on the opposite side of the warehouse experiences Wi-Fi attenuation or frame drops, its delayed `GRANT` message stalls a robot attempting to enter an aisle that is physically empty and clear[cite: 1].
* **Bandwidth Saturation on Shared Wi-Fi:** When multiple robots contend for different aisles concurrently, simultaneous fleet-wide broadcast bursts contend for the same radio spectrum, aggravating packet drop rates[cite: 1].

---

## 2. Theoretical Analysis of the First Two Approaches

### Approach 1: Quorum-Based Mutual Exclusion (Maekawa’s Algorithm)

#### Theoretical Mechanism
Rather than seeking permission from all $N-1$ robots in the warehouse, the fleet is mathematically organized into overlapping subsets called **voting sets (quorums)**, denoted as $S_i$ for each robot $i$. 

These quorums are constructed to satisfy three invariant conditions:
1. **Intersection Property:** For any two robots $i$ and $j$, their quorums must share at least one common robot ($S_i \cap S_j \neq \emptyset$).
2. **Self-Containment:** Every robot belongs to its own quorum ($i \in S_i$).
3. **Equal Sizing:** Each quorum contains roughly the same number of members, where $|S_i| = K \approx \sqrt{N}$.

When robot $i$ seeks access to a critical section, it requests permission exclusively from members of $S_i$. Because any two quorums share a common node, that arbiter node ensures that conflicting grants cannot be issued concurrently to two separate contenders.

#### Advantages
* **Sub-linear Message Complexity:** Message overhead per entry drops from $O(N)$ down to **$O(\sqrt{N})$** (typically $3\sqrt{N}$ to $5\sqrt{N}$ messages total). For a fleet of 16 robots, each machine queries roughly 4 peers instead of 15.
* **Reduced Wireless Saturation:** Eliminates full-fleet broadcast bursts, allowing robots in separate operational wings to proceed with lower channel contention.

#### Disadvantages
* **Vulnerability to Distributed Deadlock:** Unlike Ricart-Agrawala, Maekawa's algorithm permits circular wait states if members of overlapping quorums lock their votes to different requesters. Resolving this requires complex, asynchronous message extensions (`INQUIRE`, `YIELD`, `FAIL`), introducing state machine overhead and unpredictable negotiation delays.
* **Diminishing Returns on Small Fleets:** For the project’s initial scale of 3 to 5 AMRs[cite: 1], $\sqrt{N}$ is nearly equal to $N-1$, meaning the algorithm adds mathematical overhead with negligible reduction in actual message traffic.

---

### Approach 2: Geographic / Topological Scoping (Interest-Based Mutex)

#### Theoretical Mechanism
This approach applies spatial decoupling to the mutual exclusion domain. Instead of treating the warehouse as a single, uniform distributed computing state, the mutual exclusion space is partitioned along fixed physical assets identified by their pre-defined resource tags (`corridor_id`)[cite: 1].

An AMR calculates whether its rolling path intersects a critical corridor[cite: 1]. If entry is needed, the robot does not poll the global fleet[cite: 1]. Instead, it directs Ricart-Agrawala protocol primitives (`REQUEST`, `GRANT`, `ENTER`, `EXIT`)[cite: 1] solely to an **active interest set** ($C_{\text{active}}$) containing only robots that satisfy at least one condition:
1. Currently physically occupying the corridor[cite: 1].
2. Holding an unexpired lease on that `corridor_id`[cite: 1].
3. Queued or actively routing through that specific aisle threshold within their planned horizon[cite: 1].

#### Advantages
* **Constant Message Overhead Relative to Fleet Size:** Message complexity drops from $2(N_{\text{fleet}} - 1)$ to **$2(N_{\text{contenders}} - 1)$**. Because physical aisles rarely fit more than 2 or 3 contending vehicles at once, message exchanges remain bounded between 2 to 4 messages, regardless of whether the broader fleet contains 5 or 50 AMRs.
* **Zero Deadlock Risk:** Because the underlying protocol within the scoped set remains Ricart-Agrawala, total ordering via Lamport timestamps is fully preserved, guaranteeing starvation freedom and complete absence of circular deadlocks without extra recovery packets[cite: 1].
* **Direct Architectural Alignment:** The system design already defines discrete bottlenecks by `corridor_id` and tracks trajectory horizons[cite: 1]. Scoping requires no modifications to core safety layers, fallback logic, or sensor authority[cite: 1].

#### Disadvantages
* **Dynamic Membership Maintenance:** The fleet must maintain accurate interest-group tracking as robots replan or clear goals, requiring heartbeat validation or localized topic discovery to ensure no contender is omitted.
* **Boundary Consistency:** If an AMR changes its route away from an aisle without notifying the localized set, members must rely on timeout leases to purge the stale reservation[cite: 1].

---

## 3. Theoretical Comparison

| Dimension | Baseline (Ricart-Agrawala)[cite: 1] | Approach 1 (Maekawa Quorums) | Approach 2 (Geographic Scoping)[cite: 1] |
| :--- | :--- | :--- | :--- |
| **Message Complexity** | $O(N)$ — linear with fleet size[cite: 1] | $O(\sqrt{N})$ — sub-linear | $O(K)$ — bounded by local aisle contention |
| **Deadlock Vulnerability** | None (guaranteed by Lamport timestamps)[cite: 1] | High (requires `INQUIRE` / `YIELD` logic) | None (guaranteed by Lamport timestamps)[cite: 1] |
| **State Coordination** | Flat, global broadcast[cite: 1] | Fixed voting geometry matrices | Dynamic topological interest sets[cite: 1] |
| **Hardware / Scale Suitability** | Optimal only for very small fleets ($N \le 4$)[cite: 1] | High fleet counts ($N \ge 25$) | **Ideal across all fleet sizes (3 to 50+ AMRs)[cite: 1]** |