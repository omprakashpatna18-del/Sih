# AM-CORD AI × CODECO: Edge-Cloud Orchestration & Architectural Evolution

> **Reference Study**: [*A CODECO Case Study and Initial Validation for Edge Orchestration of Autonomous Mobile Robots* (arXiv:2511.08354)](https://arxiv.org/html/2511.08354v1)  
> **System Target**: AM-CORD AI Decentralized Multi-AMR Warehouse Fleet Coordination System

---

## 1. Executive Summary & Context

**CODECO (Cognitive Decentralized Edge-Cloud Orchestration)** is an open-source, edge-native Kubernetes framework (developed under European Edge-Cloud initiatives and the Eclipse Foundation) specifically created to overcome the limitations of standard cloud orchestration in mobile, resource-constrained robotic networks.

While baseline Kubernetes assumes static infrastructure, homogeneous compute, and stable high-bandwidth networks, real-world Autonomous Mobile Robots (AMRs) face **heterogeneous edge hardware** (Raspberry Pi 4, NVIDIA Jetson Nano, Hailo-8), **battery depletion**, **dynamic corridor obstacles**, and **wireless link fluctuations**.

This document outlines how the **AM-CORD AI Fleet System** integrates CODECO's data–compute–network orchestration model to unlock industrial-grade resilience, zero-downtime task handovers, and adaptive compute offloading.

---

## 2. Core Architecture: AM-CORD + CODECO Topology

```
┌──────────────────────────────────────────────────────────────────────────────────┐
│                           AM-CORD + CODECO EDGE TOPOLOGY                         │
├──────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│   ┌──────────────────────────────────────────────────────────────────────────┐   │
│   │                 WAREHOUSE EDGE CONTROLLER (K8s / K3s + CODECO)           │   │
│   │   • CAM / ACM Declarative Profiles (Rush Hour / Battery Saver / Resilient) │   │
│   │   • SWM Workload Migration Engine  • MDM Data Freshness & AoI Monitor    │   │
│   │   • Heavy Compute Offload: Cartographer Global SLAM, Fleet ML ETA        │   │
│   └──────────────────────┬────────────────────────────┬──────────────────────┘   │
│                          │ NetMA Wire-Probe           │ Zenoh L2 Overlay         │
│                          ▼                            ▼                          │
│   ┌───────────────────────────────┐          ┌───────────────────────────────┐   │
│   │  AMR 1 (Jetson Nano + Hailo)   │  P2P Sync│  AMR 2 (Raspberry Pi 4 - 8GB) │   │
│   │ • Local Costmap & 50Hz Safety │◄────────►│ • Local Costmap & 50Hz Safety │   │
│   │ • Reciprocal ORCA (Non-yield) │  (Zenoh) │ • Reciprocal ORCA (Non-yield) │   │
│   │ • CBBA Auction Node           │          │ • Low-Battery Stateful Handover│  │
│   └───────────────────────────────┘          └───────────────────────────────┘   │
└──────────────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Four Major Architectural Improvements for AM-CORD

### 1. Dynamic Workload Splitting Across Heterogeneous Edge Hardware
* **The Problem**: On low-cost embedded hardware (e.g. Raspberry Pi 4 with 8GB RAM), running concurrent high-rate 2D LiDAR costmaps, continuous SLAM, 4D WHCA* space-time search, and CBBA auctions causes compute contention and thermal throttling during peak traffic hours.
* **The CODECO Solution**:
  * **Pin Critical 50 Hz Safety Loop Onboard**: Time-critical deterministic nodes (`safety_supervisor_node`, `orca_node`, `path_follower_node`, and local costmaps) remain pinned bare-metal / containerized on the AMR.
  * **Dynamic Edge Offloading via SWM**: Heavy, non-real-time computations (Cartographer global submap generation, backward Dijkstra 4D heuristics, fleet traffic ML models) are dynamically scheduled on the Warehouse Edge Server.

---

### 2. Proactive, Stateful AMR Handover on Battery Depletion or WiFi Loss
* **The Problem**: If an AMR's battery drops below critical threshold (e.g., `< 15%`) or if an AMR loses connection in a narrow single-lane aisle, traditional systems cancel tasks or cause fleet gridlock.
* **The CODECO Solution**:
  * **PDLC (Decentralized Learning & Context)** actively models the AMR's battery discharge curve and link stability.
  * Before an AMR exhausts its battery or enters an RF dead zone, **SWM (Seamless Workload Migration)** triggers a stateful handover using Kubernetes Persistent Volume Claims (PVC) and Zenoh state sync.
  * The robot's active CBBA bundle, task waypoint queue, and localized costmap state are transferred to a healthy peer AMR in under `< 2.5 seconds` with **zero mission disruption**.

---

### 3. Data Freshness & Age of Information (AoI) for Safety Supervisors
* **The Problem**: In dense warehouse deployments, wireless packet delays can cause peer pose messages (`/peer_poses`) or trajectory intent updates (`/fleet/trajectory_intent`) to arrive late. Calculating collision cones on outdated data risks emergency stop false alarms.
* **The CODECO Solution**:
  * **MDM (Metadata Manager)** monitors data freshness and **Age of Information (AoI)**.
  * The Safety Supervisor dynamically expands its safety braking envelope $d_{\text{margin}}$ based on live MDM data latency:
    $$\text{Dynamic Safety Envelope: } d_{\text{margin}}(t) = d_{\text{base}} + v_{\text{peer}} \cdot \text{AoI}_{\text{MDM}}$$
  * This guarantees collision immunity even under transient WiFi jitter.

---

### 4. Declarative Fleet Mission Profiles via CAM
* **The Problem**: Reconfiguring fleet parameters (e.g., switching from aggressive peak throughput to low-power battery conservation) traditionally requires manual launch file restarts.
* **The CODECO Solution**:
  * Operators deploy declarative **CODECO Application Models (CAM)**:
    * **Rush-Hour Throughput Profile**: Offloads all 4D global path searches to Edge Server, relaxes battery conservation thresholds, optimizes task assignment turnaround.
    * **Energy-Conservation Profile**: Prioritizes high-battery AMRs for heavy payloads, throttles non-essential logging telemetry.
    * **Isolated Comms Fallback Profile**: Operates with pure P2P Zenoh discovery and maximum localized autonomy.

---

## 4. Key Problems Fixed & Prevented

| Identified Challenge | Traditional / Baseline Architecture | AM-CORD + CODECO Solution |
|---|---|---|
| **RPi 4 / Jetson CPU Exhaustion** | AMR drops odometry frames or delays `cmd_vel` during complex 4D re-plans. | **SWM offloads** heavy path heuristics; onboard CPU load stays under **35%**. |
| **Abrupt Battery Depletion in Aisles** | AMR halts inside a single-lane corridor, deadlocking the fleet. | **PDLC proactive migration** migrates task bundles statefully and routes the robot to docking. |
| **DDS Multicast Storms on WiFi** | High peer discovery traffic degrades real-time message delivery. | **NetMA + Zenoh (`rmw_zenohd`)** provides deterministic Layer-2 unicast discovery with 4–6 byte wire headers. |
| **Outdated Peer Intent Data** | AMR computes ORCA velocities on stale peer positions during network lag. | **MDM AoI tracking** dynamically inflates safety margins to preserve mathematical zero-collision guarantees. |

---

## 5. Implementation Roadmap for AM-CORD Fleet

### Phase 1: Microservice Containerization (Docker / K3s)
- Decouple AM-CORD into three standardized container tiers:
  1. `amcord-core-safety` (Pinned onboard: `safety_supervisor_node`, `orca_node`, `localization_node`)
  2. `amcord-coordination` (Migratable: `cbba_node`, `corridor_mutex_node`, `reservation_manager_node`)
  3. `amcord-heavy-compute` (Edge Offloadable: `whca_planner_node`, ML ETA prediction, Cartographer global SLAM)

### Phase 2: Telemetry & Prometheus Metric Exporters
- Integrate `health_node` battery telemetry and `peer_tracker_node` link metrics with CODECO MDM via Prometheus endpoints.

### Phase 3: Stateful Task Serialization over Zenoh
- Implement state checkpointing in `warehouse_tasks.py` to allow live bundle snapshots to seamlessly restore on peer robots upon migration commands.

### Phase 4: Declarative CAM Fleet Manifests
- Create declarative YAML mission profiles for warehouse operators (Peak Throughput, Night Maintenance, Low-Battery Saver).

---

## 6. Alignment with Core AI Agent Governance Rules

* **Immutable Baseline Coordination**: Proven core baseline algorithms (WHCA\* 4D space-time reservations, reciprocal ORCA velocity obstacles, CBBA decentralized auction consensus, and Corridor Mutex locking) remain the immutable coordination engine.
* **Role of CODECO**: CODECO acts purely as the **infrastructure orchestrator, telemetry observer, and edge compute offloader**, providing maximum compute headroom and high availability to the verified coordination algorithms.
