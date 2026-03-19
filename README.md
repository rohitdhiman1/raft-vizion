# raft-vizion

**A real-time interactive visualization of the Raft consensus algorithm**

Live demo: [raft-demo.rohitdhiman.net](https://raft-demo.rohitdhiman.net)

raft-vizion is a fully functional implementation of the Raft distributed consensus protocol with an interactive web-based visualization. Watch leader elections unfold, observe log replication in real time, inject network partitions, and see how the cluster recovers — all from your browser.

Built to make distributed consensus *visible* and *tangible*.

---

## Why this project exists

Raft is the consensus algorithm behind systems like etcd, CockroachDB, and TiKV. Understanding it deeply is essential for anyone working with distributed systems, Kubernetes, or fault-tolerant infrastructure.

Most learning resources explain Raft with static diagrams. This project lets you **operate** a Raft cluster: kill nodes, partition networks, submit log entries, and watch the protocol handle every scenario in real time.

---

## What you can do

- **Watch leader election** — see election timers tick down, vote requests fly between nodes, and a new leader emerge with majority consensus
- **Observe log replication** — submit entries via the UI and watch them propagate from leader to followers, with commit confirmation once a majority acknowledges
- **Inject network partitions** — isolate one or more nodes and observe split-brain prevention (the minority partition cannot elect a leader because it lacks quorum)
- **Kill and restart nodes** — take down the leader and watch the cluster detect the failure via heartbeat timeout, then elect a new leader within seconds
- **Heal the network** — restore partitioned nodes and watch them catch up on missed log entries
- **Slow-motion mode** — reduce the simulation speed to trace every RPC message step by step
- **Event timeline** — a scrollable log of every significant event (elections, votes, commits, failures) with timestamps

---

## Architecture

```
┌─────────────────────────────────────────────────┐
│                React frontend                    │
│         (Cloudflare Pages / Static)              │
│                                                  │
│  ┌──────────┐ ┌──────────┐ ┌──────────────────┐ │
│  │ Cluster  │ │ Log      │ │ Event timeline   │ │
│  │ view     │ │ compare  │ │ & controls       │ │
│  └──────────┘ └──────────┘ └──────────────────┘ │
└──────────────────┬──────────────────────────────┘
                   │ WebSocket
┌──────────────────┴──────────────────────────────┐
│              WebSocket gateway                    │
│         (Spring Boot / Netty)                    │
│  Streams cluster state to connected clients      │
└──────────────────┬──────────────────────────────┘
                   │
┌──────────────────┴──────────────────────────────┐
│              Raft cluster (5 nodes)               │
│                                                  │
│  ┌───────┐  ┌───────┐  ┌───────┐               │
│  │Node 1 │──│Node 2 │──│Node 3 │               │
│  └───┬───┘  └───────┘  └───┬───┘               │
│      │   ┌───────┐  ┌─────┘                     │
│      └───│Node 4 │──│Node 5 │                   │
│          └───────┘  └───────┘                    │
│                                                  │
│  Each node: independent state machine,           │
│  election timer, persistent log, term counter    │
│  Communication: gRPC between nodes               │
└─────────────────────────────────────────────────┘
```

---

## Tech stack

| Layer | Technology | Purpose |
|-------|-----------|---------|
| Raft nodes | Java 21 + Spring Boot | Core consensus implementation with gRPC inter-node communication |
| WebSocket gateway | Spring Boot + WebSocket | Streams real-time cluster state to the frontend |
| Frontend | React + TypeScript | Interactive cluster visualization with animated message passing |
| Containerisation | Docker + Docker Compose | Each Raft node runs in its own container, simulating true process isolation |
| Hosting (backend) | Oracle Cloud free tier | ARM instance running the full Docker Compose stack |
| Hosting (frontend) | Cloudflare Pages | Static React build served globally, connected to backend via Cloudflare Tunnel |

---

## Project structure

```
raft-vizion/
├── docker-compose.yml
├── README.md
│
├── raft-core/                    # Java — Raft protocol implementation
│   ├── src/main/java/
│   │   └── net/rohitdhiman/raft/
│   │       ├── RaftNode.java              # Main node lifecycle
│   │       ├── RaftState.java             # Follower / Candidate / Leader states
│   │       ├── Log.java                   # Replicated log with persistence
│   │       ├── ElectionManager.java       # Election timers and vote handling
│   │       ├── ReplicationManager.java    # Log replication to followers
│   │       ├── StateMachine.java          # Committed entry application
│   │       ├── rpc/
│   │       │   ├── RequestVoteRpc.java
│   │       │   ├── AppendEntriesRpc.java
│   │       │   └── RaftGrpcService.java
│   │       └── config/
│   │           └── ClusterConfig.java
│   ├── src/test/java/                     # Unit + integration tests
│   ├── Dockerfile
│   └── pom.xml
│
├── gateway/                      # Java — WebSocket gateway
│   ├── src/main/java/
│   │   └── net/rohitdhiman/gateway/
│   │       ├── GatewayApplication.java
│   │       ├── ClusterStateAggregator.java
│   │       ├── WebSocketHandler.java
│   │       └── FaultInjectionController.java  # REST API for kill/partition/heal
│   ├── Dockerfile
│   └── pom.xml
│
├── frontend/                     # React — interactive visualization
│   ├── src/
│   │   ├── App.tsx
│   │   ├── components/
│   │   │   ├── ClusterView.tsx            # Node circles with animated heartbeats
│   │   │   ├── LogComparisonView.tsx      # Side-by-side log tables per node
│   │   │   ├── EventTimeline.tsx          # Scrollable event log
│   │   │   ├── ControlPanel.tsx           # Kill / partition / heal / speed controls
│   │   │   └── MessageAnimation.tsx       # Animated RPCs between nodes
│   │   ├── hooks/
│   │   │   └── useClusterWebSocket.ts     # WebSocket connection + state management
│   │   └── types/
│   │       └── raft.ts                    # TypeScript types for cluster state
│   ├── Dockerfile
│   └── package.json
│
├── chaos/                        # Python — chaos testing scripts
│   ├── scenarios/
│   │   ├── leader_kill.py
│   │   ├── network_partition.py
│   │   ├── cascading_failure.py
│   │   └── split_brain.py
│   ├── requirements.txt
│   └── README.md
│
└── docs/
    ├── architecture.md
    ├── raft-protocol.md           # How Raft works + how this impl maps to the paper
    ├── design-decisions.md        # ADRs: why gRPC, why not Akka, etc.
    └── chaos-test-results.md      # Results from chaos scenarios with screenshots
```

---

## Raft implementation details

This implementation follows the Raft paper (Ongaro & Ousterhout, 2014) with the following scope:

### Implemented

- **Leader election** — randomised election timeouts (150–300ms), `RequestVote` RPC, majority-based leader selection, pre-vote protocol to prevent disruptions from partitioned nodes rejoining
- **Log replication** — `AppendEntries` RPC, log consistency check (prevLogIndex + prevLogTerm), backtracking on conflict, commit index advancement on majority acknowledgement
- **Safety** — election restriction (candidates must have up-to-date logs), leader completeness (committed entries are never lost), state machine safety (entries applied in order)
- **Cluster membership** — fixed 5-node cluster (sufficient for demonstrating all failure scenarios)
- **Persistence** — each node persists `currentTerm`, `votedFor`, and log entries to disk, surviving restarts

### Not implemented (out of scope for this demo)

- Dynamic membership changes (joint consensus)
- Log compaction / snapshotting
- Client session semantics (linearisability)
- Read-only optimisation (lease-based reads)

---

## Running locally

### Prerequisites

- Docker and Docker Compose
- Java 21+ (for local development without Docker)
- Node.js 20+ (for frontend development)

### Quick start

```bash
git clone https://github.com/rohitdhiman/raft-vizion.git
cd raft-vizion

# Start the full stack
docker compose up --build

# Frontend: http://localhost:3000
# Gateway API: http://localhost:8080
# Node endpoints: localhost:9001–9005
```

### Development mode

```bash
# Terminal 1 — start backend (Raft nodes + gateway)
docker compose up raft-node-1 raft-node-2 raft-node-3 raft-node-4 raft-node-5 gateway

# Terminal 2 — start frontend with hot reload
cd frontend && npm install && npm run dev
```

---

## API reference

### WebSocket (ws://localhost:8080/ws/cluster)

Streams JSON cluster state every 200ms:

```json
{
  "term": 3,
  "nodes": [
    {
      "id": 1,
      "state": "leader",
      "term": 3,
      "commitIndex": 12,
      "lastApplied": 12,
      "log": [
        { "index": 1, "term": 1, "command": "set x 1" }
      ],
      "alive": true,
      "partitioned": false
    }
  ],
  "messages": [
    {
      "from": 1,
      "to": 2,
      "type": "AppendEntries",
      "term": 3,
      "timestamp": 1711900800000
    }
  ],
  "events": [
    {
      "type": "LEADER_ELECTED",
      "nodeId": 1,
      "term": 3,
      "timestamp": 1711900800000,
      "detail": "Node 1 elected with 4 votes"
    }
  ]
}
```

### REST — fault injection

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/nodes/{id}/kill` | POST | Stops a node (simulates crash) |
| `/api/nodes/{id}/restart` | POST | Restarts a stopped node |
| `/api/nodes/{id}/partition` | POST | Isolates a node from the network |
| `/api/network/heal` | POST | Removes all network partitions |
| `/api/cluster/submit` | POST | Submits a log entry to the leader |
| `/api/cluster/state` | GET | Returns current cluster state (snapshot) |
| `/api/config/speed` | PUT | Sets simulation speed multiplier (0.25x–4x) |

---

## Chaos testing

The `chaos/` directory contains scripted failure scenarios that exercise the cluster and record results:

| Scenario | What it tests |
|----------|--------------|
| `leader_kill.py` | Kill the leader, verify a new leader is elected within the expected timeout, verify no committed entries are lost |
| `network_partition.py` | Partition 2 nodes from 3, verify the majority partition elects a leader, verify the minority partition does not |
| `cascading_failure.py` | Kill nodes one by one until quorum is lost, then restore them and verify recovery |
| `split_brain.py` | Create a symmetric partition (2-1-2), verify at most one partition can elect a leader |

Run all scenarios:

```bash
cd chaos
pip install -r requirements.txt
python -m pytest scenarios/ -v
```

Results with screenshots are documented in `docs/chaos-test-results.md`.

---

## Deployment

### Backend — Oracle Cloud free tier

```bash
# SSH into your Oracle Cloud ARM instance
ssh ubuntu@your-instance-ip

# Clone and start
git clone https://github.com/rohitdhiman/raft-vizion.git
cd raft-vizion
docker compose -f docker-compose.prod.yml up -d

# Install Cloudflare Tunnel
cloudflared tunnel create raft-vizion
cloudflared tunnel route dns raft-vizion raft-api.rohitdhiman.net
cloudflared tunnel run raft-vizion
```

### Frontend — Cloudflare Pages

Connect the GitHub repo to Cloudflare Pages:

- Build command: `cd frontend && npm run build`
- Output directory: `frontend/dist`
- Environment variable: `VITE_WS_URL=wss://raft-api.rohitdhiman.net/ws/cluster`

The frontend auto-deploys on every push to `main`.

---

## What I learned

Building this project taught me several things that reading the Raft paper alone would not have:

- **Election timer tuning matters enormously.** Too tight and you get election storms after partitions heal. Too loose and failover is slow. The randomisation range (150–300ms in the paper) is a carefully chosen sweet spot.
- **Log consistency backtracking is where bugs hide.** The paper's description is precise, but implementing the "decrement nextIndex and retry" loop correctly — especially when a follower has divergent entries from a previous term — required careful testing.
- **Observability changes how you debug distributed systems.** The WebSocket-based state streaming I built for the frontend turned out to be the most valuable debugging tool during development. Being able to *see* every message and state transition made finding bugs dramatically faster.
- **Network partitions are harder to reason about than crashes.** A crashed node is cleanly absent. A partitioned node is alive but unreachable — and it might be running elections, incrementing terms, and causing disruption when it rejoins.

---

## References

- [In Search of an Understandable Consensus Algorithm](https://raft.github.io/raft.pdf) — the original Raft paper by Diego Ongaro and John Ousterhout
- [The Secret Lives of Data — Raft](http://thesecretlivesofdata.com/raft/) — an excellent interactive walkthrough of the algorithm
- [raft.github.io](https://raft.github.io/) — reference implementations, TLA+ spec, and community resources
- [etcd/raft](https://github.com/etcd-io/raft) — production Raft implementation in Go (used by Kubernetes)

---

## License

MIT

---

## Author

**Rohit Dhiman** — [rohitdhiman.net](https://rohitdhiman.net)