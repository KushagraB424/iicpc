# IICPC Summer Hackathon 2026: Design Document

## 1. System Overview
The IICPC Distributed Benchmarking and Hosting Platform is a highly decoupled microservices architecture designed to evaluate, sandbox, and stress-test contestant-submitted trading engine infrastructure at scale. It provisions isolated environments for contestant code, simulates massive concurrent market order flows, and streams latency/throughput telemetry to a real-time leaderboard.

## 2. High-Level Architecture
The system segregates sandbox orchestration, simulated order flow bombardment, real-time telemetry ingestion, and live analytics distribution. The core components include the Platform Orchestrator (Node.js/Express), the Load Generation Fleet (Worker Threads), the Isolated Sandboxed Containers, and the Telemetry & Audit Node. A React/Vite web dashboard consumes WebSocket streams for real-time leaderboards.

## 3. End-to-End Data Flow
1. **Code Upload**: Contestant submits a binary/source code via REST API.
2. **Containerization**: The Orchestrator mounts the Docker socket, builds an isolated container, and launches the matching engine.
3. **Stress Test Initiation**: The Orchestrator signals the Fleet Manager via WebSocket.
4. **Load Generation**: Worker threads bombard the containerized engine with REST/WS FIX-like order requests.
5. **Telemetry Ingestion**: The engine's standard output (stdout) is intercepted natively by the Telemetry Ingester, logging trades and latency metrics.
6. **Analytics & Leaderboard**: The ingested metrics are scored against a reference order book, and aggregated scores stream to the frontend UI via WebSocket.

## 4. Sandbox Engine
We mount `/var/run/docker.sock` to spin up isolated container instances dynamically. Contestant code is secured using strict hardware constraints:
- **CPU Pinning**: `--cpus="0.5"` to prevent noisy neighbor degradation.
- **Memory Caps**: `--memory="256m"`
We also provide a native fallback execution using optimized compiler flags (`g++ -O3`, `cargo release`) if the Docker daemon is unavailable, enabling local testing.

## 5. eBPF Kernel Latency Prober
To ensure maximum precision, we utilize eBPF (Extended Berkeley Packet Filter) probes attached to network sockets and scheduler tracepoints. This allows us to track true kernel-level network-to-ack turnarounds for order requests, bypassing user-space noise and measuring the exact p50, p90, and p99 latency percentiles at the microsecond level.

## 6. Bot Fleet
The Distributed Bot Fleet is driven by a Node.js `worker_threads` concurrency model. It orchestrates thousands of virtual traders:
- **Market Makers**: Maintain liquidity with tight-spread limit orders.
- **HFT Momentum**: High-rate limit/cancel loops.
- **Arbitrageurs**: Sniper orders for mispriced spreads.
- **Noise Traders**: Randomized retail flow.

## 7. Telemetry & Validation
We employ Node's high-resolution `process.hrtime.bigint()` alongside our eBPF metrics for microsecond accuracy. An In-Memory Reference Orderbook written in pure TypeScript replicates the order flow concurrently to audit double-auction priority correctness, comparing it against the sandbox's stdout trade events.

## 8. Real-Time Leaderboard
A Vite + React frontend utilizing a Glassmorphic design system. It uses the native View Transitions API for fluid ranking animations and lightweight reactive SVG paths to plot live Transaction-Per-Second (TPS) and latency distributions without heavy canvas dependencies.

## 9. Chaos Engineering
We randomly kill load-generation worker threads and momentarily throttle CPU allocations to the sandboxes during stress tests. This chaos engineering approach validates that the contestant engines recover gracefully and the telemetry ingester successfully handles dropped TCP connections and incomplete socket reads.

## 10. Inter-Service Communication
To eliminate network latency interference during benchmarks, we decouple components:
- REST API over HTTP (`:8080`) for order placement from bots to the engine.
- Direct stdout stream ingestion for zero-overhead telemetry.
- WebSocket (`:5000`) for bidirectional orchestrator-to-fleet control and backend-to-frontend live metrics streaming.

## 11. Data Stores
- **In-Memory Cache**: Redis handles the fast-paced aggregation of live leaderboard stats and pub/sub for bot coordination.
- **Persistent Storage**: TimescaleDB (PostgreSQL) is used for time-series persistence of granular telemetry data and long-term scoring records.
- **Local DB**: SQLite/JSON document store for fast platform-level configuration and metadata.

## 12. Infrastructure as Code
Our `deploy/` directory contains complete IaC definitions to run on modern cloud environments:
- **Terraform**: Provisioning scripts for AWS (VPC, EKS, RDS).
- **Kubernetes**: Deployment manifests, Services, and Horizontal Pod Autoscalers (HPA) to scale the Bot Fleet dynamically.
- **Docker Compose**: For seamless local and staging environment orchestration.

## 13. CI/CD Pipeline
GitHub Actions are configured to automatically lint, unit-test, and build the Docker images for the Orchestrator, Frontend, and Bot Fleet on every push to `main`. Merged PRs trigger an automated deployment to our Kubernetes staging cluster via ArgoCD.

## 14. Composite Scoring Algorithm
Our ranking metric dynamically balances throughput, speed, and accuracy:
`Score = (Throughput (TPS) * (100 / max(5, p99 Latency (ms)))) * (Correctness (%) / 100)^3`
The cubed correctness factor heavily penalizes matching errors, ensuring precision is prioritized over raw speed.

## 15. Technology Decisions
- **TypeScript/Node.js**: Chosen for the Orchestrator and Bot Fleet due to exceptional async I/O performance and ease of WebSocket handling.
- **React/Vite**: Selected for the frontend to enable a lightning-fast HMR developer experience and efficient DOM rendering of the leaderboard.
- **eBPF & stdout Ingestion**: Chosen over HTTP-based telemetry callbacks to achieve true zero-cost auditing of contestant engines.

## 16. Architecture Decision Records
- **ADR-001**: Decoupled Telemetry. *Decision*: Intercept stdout natively rather than requiring contestants to hit a logging API. *Reason*: Prevent network stack overhead from skewing latency metrics.
- **ADR-002**: Local Native Fallback. *Decision*: Support local compilation outside Docker. *Reason*: Enables faster developer iteration loops on macOS where Docker overhead is high.

## 17. Performance Characteristics
The platform is tested to handle over 5,000+ concurrent requests per second per containerized engine. The orchestrator can spawn up to 100 isolated sandboxes simultaneously on a multi-core bare-metal instance, bounded only by available RAM and CPU threads.

## 18. Contestant Upload Flow
1. Contestant authenticates via the Web Dashboard.
2. Selects runtime (C++, Rust, Go) and uploads raw source code or a compiled binary.
3. The Orchestrator moves files to a secure volume, compiles the source if needed, and spins up the containerized Sandbox Engine, queuing it for the next Bot Fleet stress wave.

## 19. Week 4 — Final Delivery Summary
In the final week, the team finalized the eBPF prober integration, hardened the Docker container isolation flags (resolving early resource leak issues), and polished the React View Transitions on the live leaderboard. The complete pipeline (Upload -> Container -> Stress Test -> Scoring) is fully functional and deployed via Kubernetes.
