# llm-d Architecture Diagrams

## 1. System Components Diagram

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              Client Applications                             │
│                    (Chat, RAG, Agents, Multimodal, etc.)                    │
└────────────────────────────────┬────────────────────────────────────────────┘
                                 │ HTTP/HTTPS Requests
                                 │ (OpenAI Compatible API)
                                 ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                         Kubernetes Gateway API                               │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │                      Gateway Controller                              │   │
│  │        (Istio / Kong Gateway / GKE External LB)                     │   │
│  └────────────────────────────┬─────────────────────────────────────────┘   │
└────────────────────────────────┼─────────────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                     llm-d Inference Scheduler                                │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │                      Envoy Proxy Layer                               │   │
│  │  ┌───────────────────────────────────────────────────────────────┐   │   │
│  │  │  • Model Routing & Version Rollout                           │   │   │
│  │  │  • Flow Control & Rate Limiting                              │   │   │
│  │  │  • Request Filtering & Scoring                               │   │   │
│  │  └───────────────────────────────────────────────────────────────┘   │   │
│  │                                                                         │   │
│  │  ┌───────────────────────────────────────────────────────────────┐   │   │
│  │  │  Intelligent Load Balancing Policies:                        │   │   │
│  │  │  • Load-Aware Balancing                                       │   │   │
│  │  │  • Prefix-Cache-Aware Routing (Approximate/Precise)          │   │   │
│  │  │  • Predicted Latency Balancing (Experimental)                │   │   │
│  │  │  • SLA-Aware Scheduling                                       │   │   │
│  │  │  • P/D-Aware Routing (for disaggregation)                    │   │   │
│  │  └───────────────────────────────────────────────────────────────┘   │   │
│  └────────────────────────────┬─────────────────────────────────────────┘   │
└────────────────────────────────┼─────────────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                        vLLM Model Server Pool                                │
│                                                                               │
│  ┌─────────────────────┐  ┌─────────────────────┐  ┌─────────────────────┐  │
│  │  Prefill Workers    │  │  Decode Workers     │  │  Mixed Workers      │  │
│  │                     │  │                     │  │                     │  │
│  │  • TP=1, DP=4      │  │  • TP=4, DP=1      │  │  • TP=2, DP=2      │  │
│  │  • High throughput  │  │  • Low latency     │  │  • Balanced        │  │
│  │  • Compute-bound    │  │  • Memory-bound    │  │  • General use     │  │
│  │                     │  │                     │  │                     │  │
│  │  ┌───────────────┐  │  │  ┌───────────────┐  │  │  ┌───────────────┐  │  │
│  │  │ vLLM Engine   │  │  │  │ vLLM Engine   │  │  │  │ vLLM Engine   │  │  │
│  │  │ + KV Cache    │  │  │  │ + KV Cache    │  │  │  │ + KV Cache    │  │  │
│  │  └───────────────┘  │  │  └───────────────┘  │  │  └───────────────┘  │  │
│  │                     │  │                     │  │                     │  │
│  │  ┌───────────────┐  │  │  ┌───────────────┐  │  │  ┌───────────────┐  │  │
│  │  │ P/D Sidecar   │  │  │  │ P/D Sidecar   │  │  │  │ Monitoring    │  │  │
│  │  │ (for disagg)  │  │  │  │ (for disagg)  │  │  │  │ Sidecar       │  │  │
│  │  └───────────────┘  │  │  └───────────────┘  │  │  └───────────────┘  │  │
│  └─────────────────────┘  └─────────────────────┘  └─────────────────────┘  │
│                                                                               │
│  ────────────────── KV Cache Transfer (NIXL/RDMA) ──────────────────────    │
│                                                                               │
└─────────────────────────────────────────────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                    Prefix Cache Hierarchy (Optional)                         │
│                                                                               │
│  ┌─────────────────────┐  ┌─────────────────────┐  ┌─────────────────────┐  │
│  │  GPU HBM Cache      │  │  Host Memory Cache  │  │  Remote Storage     │  │
│  │  (per-instance)     │  │  (local offload)    │  │  (shared/LMCache)   │  │
│  └─────────────────────┘  └─────────────────────┘  └─────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                          Infrastructure Layer                                │
│                                                                               │
│  ┌─────────────────────┐  ┌─────────────────────┐  ┌─────────────────────┐  │
│  │  Kubernetes 1.29+   │  │  Accelerator HW     │  │  Fast Networking    │  │
│  │  • Pods/StatefulSet │  │  • NVIDIA H100/A100 │  │  • RDMA/InfiniBand  │  │
│  │  • LeaderWorkerSet  │  │  • AMD MI250+       │  │  • RoCE             │  │
│  │  • Services/Ingress │  │  • Intel XPU        │  │  • NVLink/TPU ICI   │  │
│  │  • HPA/VPA          │  │  • Google TPU v5e+  │  │  • AWS EFA          │  │
│  └─────────────────────┘  └─────────────────────┘  └─────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                     Observability & Monitoring                               │
│                                                                               │
│  • Prometheus Metrics (PodMonitor)     • Grafana Dashboards                 │
│  • OpenTelemetry Traces                • Log Aggregation                     │
│  • Custom vLLM Metrics                 • Performance Benchmarking (guidellm) │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Component Details

#### 1. **Inference Scheduler (IGW-based)**
- **Technology**: Kubernetes Gateway API + Envoy Proxy
- **Repository**: `llm-d/llm-d-inference-scheduler`
- **Key Features**:
  - Model routing and version management
  - Custom load balancing algorithms
  - Prefix cache awareness
  - P/D disaggregation coordination
  - Flow control and rate limiting

#### 2. **vLLM Model Servers**
- **Technology**: vLLM inference engine
- **Repository**: `vllm-project/vllm` (upstream)
- **Deployment Modes**:
  - Single-host (standard)
  - Multi-host via Ray (for TP > 8)
  - Multi-host via LeaderWorkerSet (for expert parallelism)
- **Custom Kernels**:
  - `pplx` - All2All backend
  - `deepep` - DeepSeek kernels (high throughput/low latency)
  - `deepgemm` - Custom GEMM operations

#### 3. **KV Cache Transfer**
- **Technology**: NIXL (Network Inference eXtension Library)
- **Repository**: `ai-dynamo/nixl`
- **Supported Transports**:
  - RDMA over InfiniBand
  - RDMA over RoCE
  - TPU ICI (Inter-Chip Interconnect)
  - TCP fallback with UCX

#### 4. **Orchestration**
- **Kubernetes Resources**:
  - StatefulSets for stateful workloads
  - LeaderWorkerSet for multi-node model servers
  - Gateway/HTTPRoute for traffic management
  - InferencePool CRD for model server groups
  - HPA for autoscaling

---

## 2. Request Call Flow Diagrams

### 2.1 Standard Inference Flow (No Disaggregation)

```
┌─────────┐
│ Client  │
└────┬────┘
     │ 1. POST /v1/chat/completions
     ▼
┌────────────────────┐
│ Gateway Controller │
│   (Istio/Kong)     │
└────────┬───────────┘
         │ 2. Route to InferencePool
         ▼
┌────────────────────────────────┐
│  Inference Scheduler (Envoy)   │
│  ┌──────────────────────────┐  │
│  │ Request Analysis         │  │
│  │ • Extract prompt         │  │
│  │ • Compute prefix hash    │  │
│  │ • Check request metadata │  │
│  └─────────┬────────────────┘  │
│            │ 3. Score backends  │
│  ┌─────────▼────────────────┐  │
│  │ Backend Scoring          │  │
│  │ • Load-aware score       │  │
│  │ • Cache-aware score      │  │
│  │ • Latency prediction     │  │
│  └─────────┬────────────────┘  │
│            │ 4. Select best     │
│  ┌─────────▼────────────────┐  │
│  │ Backend Selection        │  │
│  │ • Pick highest score     │  │
│  │ • Apply flow control     │  │
│  └─────────┬────────────────┘  │
└────────────┼────────────────────┘
             │ 5. Forward request
             ▼
┌────────────────────────────────┐
│      vLLM Model Server         │
│  ┌──────────────────────────┐  │
│  │ Prefix Cache Lookup      │  │
│  │ • Check HBM cache        │  │
│  │ • Check host memory      │  │
│  └─────────┬────────────────┘  │
│            │                    │
│  ┌─────────▼────────────────┐  │
│  │ Prefill Phase            │  │
│  │ • Process prompt tokens  │  │
│  │ • Generate KV cache      │  │
│  │ • Store in cache         │  │
│  └─────────┬────────────────┘  │
│            │                    │
│  ┌─────────▼────────────────┐  │
│  │ Decode Phase             │  │
│  │ • Generate tokens        │  │
│  │ • Stream response        │  │
│  └─────────┬────────────────┘  │
└────────────┼────────────────────┘
             │ 6. Stream tokens
             ▼
┌────────────────────┐
│  Inference         │
│  Scheduler         │
└────────┬───────────┘
         │ 7. Stream to client
         ▼
┌─────────────┐
│   Client    │
│  (receives  │
│   tokens)   │
└─────────────┘
```

### 2.2 Disaggregated Serving Flow (P/D Split)

```
┌─────────┐
│ Client  │
└────┬────┘
     │ 1. POST /v1/chat/completions
     ▼
┌──────────────────────────────────────────────────────────┐
│            Inference Scheduler (Envoy)                   │
│  ┌────────────────────────────────────────────────────┐  │
│  │ P/D-Aware Request Analysis                         │  │
│  │ • Classify request (prefill vs decode)             │  │
│  │ • Estimate ISL (input sequence length)             │  │
│  │ • Select P or D worker pool                        │  │
│  └──────────────────┬─────────────────────────────────┘  │
└─────────────────────┼────────────────────────────────────┘
                      │
        ┌─────────────┴─────────────┐
        │ 2a. Route to Prefill      │ 2b. Route to Decode
        │     Worker Pool           │     Worker Pool (for
        │     (for new requests)    │     continuation)
        ▼                           ▼
┌────────────────────────┐   ┌────────────────────────┐
│  Prefill vLLM Server   │   │  Decode vLLM Server    │
│  (TP=1, DP=4)          │   │  (TP=4, DP=1)          │
│  ┌──────────────────┐  │   │  ┌──────────────────┐  │
│  │ Process Prompt   │  │   │  │ Waiting for KV   │  │
│  │ • Compute tokens │  │   │  │ cache transfer   │  │
│  │ • Generate KV    │  │   │  └──────────────────┘  │
│  │   cache blocks   │  │   │                        │
│  └────────┬─────────┘  │   │                        │
│           │ 3. KV ready│   │                        │
│  ┌────────▼─────────┐  │   │                        │
│  │ P/D Sidecar      │  │   │  ┌──────────────────┐  │
│  │ • Prepare KV     │  │   │  │ P/D Sidecar      │  │
│  │   transfer       │  │   │  │ • Receive KV     │  │
│  │ • Notify D       │  │   │  │   blocks via     │  │
│  │   worker         │  │   │  │   NIXL           │  │
│  └────────┬─────────┘  │   │  └────────┬─────────┘  │
└───────────┼────────────┘   └───────────┼────────────┘
            │                            │
            │ 4. KV Cache Transfer       │
            │    (NIXL over RDMA/RoCE)   │
            └────────────────────────────┤
                                         │
                             ┌───────────▼───────────┐
                             │  Decode vLLM Server   │
                             │  ┌──────────────────┐ │
                             │  │ Decode Phase     │ │
                             │  │ • Receive KV     │ │
                             │  │ • Generate       │ │
                             │  │   output tokens  │ │
                             │  │ • Stream result  │ │
                             │  └────────┬─────────┘ │
                             └───────────┼───────────┘
                                         │ 5. Stream tokens
                                         ▼
                             ┌───────────────────────┐
                             │ Inference Scheduler   │
                             └───────────┬───────────┘
                                         │ 6. Return to client
                                         ▼
                                   ┌─────────┐
                                   │ Client  │
                                   └─────────┘
```

### 2.3 Prefix-Cache-Aware Routing Flow

```
┌─────────┐
│ Client  │ (sends request with shared prefix)
└────┬────┘
     │ 1. Request: "Explain quantum computing..."
     ▼
┌────────────────────────────────────────────────────────┐
│         Inference Scheduler                            │
│  ┌──────────────────────────────────────────────────┐  │
│  │ Prefix Analysis                                  │  │
│  │ • Extract prompt prefix                          │  │
│  │ • Compute hash: hash("Explain quantum")          │  │
│  │ • prefix_hash = 0xABCD1234                       │  │
│  └─────────────────┬────────────────────────────────┘  │
│                    │ 2. Query backends                 │
│  ┌─────────────────▼────────────────────────────────┐  │
│  │ Backend Scoring (Prefix-Cache-Aware)            │  │
│  │                                                  │  │
│  │ Backend A: load=0.6, prefix_hit=YES  → score=90 │  │
│  │ Backend B: load=0.4, prefix_hit=NO   → score=40 │  │
│  │ Backend C: load=0.3, prefix_hit=YES  → score=95 │  │
│  │                                                  │  │
│  │ ✓ Select Backend C (highest score)              │  │
│  └─────────────────┬────────────────────────────────┘  │
└────────────────────┼────────────────────────────────────┘
                     │ 3. Route to Backend C
                     ▼
         ┌───────────────────────────┐
         │  vLLM Server C            │
         │  ┌─────────────────────┐  │
         │  │ Prefix Cache Lookup │  │
         │  │ hash=0xABCD1234     │  │
         │  │ → CACHE HIT! ✓      │  │
         │  └──────────┬──────────┘  │
         │             │ 4. Reuse KV  │
         │  ┌──────────▼──────────┐  │
         │  │ Skip prefill for    │  │
         │  │ cached tokens       │  │
         │  │ Process only new:   │  │
         │  │ "...in simple terms"│  │
         │  └──────────┬──────────┘  │
         │             │ 5. Fast decode│
         │  ┌──────────▼──────────┐  │
         │  │ Generate output     │  │
         │  │ (reduced TTFT)      │  │
         │  └──────────┬──────────┘  │
         └─────────────┼──────────────┘
                       │ 6. Stream result
                       ▼
                 ┌─────────┐
                 │ Client  │ (faster response!)
                 └─────────┘

Benefits:
• Reduced Time-to-First-Token (TTFT)
• Higher throughput (skip redundant compute)
• Better GPU utilization
```

### 2.4 Wide Expert Parallelism Flow (MoE Models)

```
┌─────────┐
│ Client  │ Request for DeepSeek-R1 (MoE model)
└────┬────┘
     │ 1. POST /v1/chat/completions
     ▼
┌────────────────────────────────┐
│  Inference Scheduler           │
└────────┬───────────────────────┘
         │ 2. Route to LeaderWorkerSet
         ▼
┌─────────────────────────────────────────────────────────────┐
│            DeepSeek-R1 LeaderWorkerSet                      │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │              Leader Pod (TP=8, EP=4)                │   │
│  │  ┌───────────────────────────────────────────────┐  │   │
│  │  │ Request Distribution                          │  │   │
│  │  │ • Receives inference request                  │  │   │
│  │  │ • Coordinates expert routing                  │  │   │
│  │  │ • Manages worker communication                │  │   │
│  │  └─────────────────┬─────────────────────────────┘  │   │
│  └────────────────────┼────────────────────────────────┘   │
│                       │ 3. Distribute to workers            │
│                       │    (via All2All collective)         │
│       ┌───────────────┼───────────────┬────────────────┐    │
│       │               │               │                │    │
│  ┌────▼─────┐   ┌────▼─────┐   ┌────▼─────┐   ┌─────▼───┐ │
│  │ Worker 0 │   │ Worker 1 │   │ Worker 2 │   │Worker 3 │ │
│  │          │   │          │   │          │   │         │ │
│  │ Expert   │   │ Expert   │   │ Expert   │   │ Expert  │ │
│  │ Shard 0  │   │ Shard 1  │   │ Shard 2  │   │ Shard 3 │ │
│  │          │   │          │   │          │   │         │ │
│  │ ┌──────┐ │   │ ┌──────┐ │   │ ┌──────┐ │   │┌──────┐ │ │
│  │ │Expert│ │   │ │Expert│ │   │ │Expert│ │   ││Expert│ │ │
│  │ │ 0-31 │ │   │ │32-63 │ │   │ │64-95 │ │   ││96-127│ │ │
│  │ └──────┘ │   │ └──────┘ │   │ └──────┘ │   │└──────┘ │ │
│  │          │   │          │   │          │   │         │ │
│  │ Process  │   │ Process  │   │ Process  │   │ Process │ │
│  │ tokens   │   │ tokens   │   │ tokens   │   │ tokens  │ │
│  │ with     │   │ with     │   │ with     │   │ with    │ │
│  │ experts  │   │ experts  │   │ experts  │   │ experts │ │
│  └────┬─────┘   └────┬─────┘   └────┬─────┘   └─────┬───┘ │
│       │              │              │               │      │
│       │ 4. All2All communication (NVLink/RDMA)      │      │
│       │              │              │               │      │
│       └──────────────┴──────────────┴───────────────┘      │
│                       │ 5. Gather results                   │
│  ┌────────────────────▼────────────────────────────────┐   │
│  │              Leader Pod                             │   │
│  │  • Aggregates expert outputs                        │   │
│  │  • Applies attention layers                         │   │
│  │  • Generates final output                           │   │
│  └────────────────────┬────────────────────────────────┘   │
└─────────────────────────┼──────────────────────────────────┘
                          │ 6. Return response
                          ▼
                    ┌─────────┐
                    │ Client  │
                    └─────────┘

Key Features:
• Expert Parallelism (EP=4): 128 experts sharded across 4 workers
• Tensor Parallelism (TP=8): Each expert uses 8 GPUs
• All2All collective for efficient expert routing
• Achieves 2.2k tokens/s/gpu on H200
```

---

## 3. Data Flow Patterns

### 3.1 KV Cache Transfer (NIXL)

```
Prefill Worker                         Decode Worker
┌──────────────┐                      ┌──────────────┐
│   vLLM       │                      │   vLLM       │
│   Engine     │                      │   Engine     │
└──────┬───────┘                      └──────▲───────┘
       │ 1. KV blocks ready                  │ 5. Receive KV
       ▼                                     │
┌──────────────┐                      ┌──────────────┐
│ P/D Sidecar  │                      │ P/D Sidecar  │
│              │                      │              │
│ • Serialize  │                      │ • Deserialize│
│   KV blocks  │                      │   KV blocks  │
│ • Register   │                      │ • Allocate   │
│   buffers    │                      │   memory     │
└──────┬───────┘                      └──────▲───────┘
       │ 2. NIXL send                        │ 4. NIXL recv
       ▼                                     │
┌──────────────────────────────────────────────────────┐
│                  NIXL Library                        │
│  ┌──────────────┐         ┌──────────────┐          │
│  │ UCX/IB Stack │         │ GPU Direct   │          │
│  │ • RDMA Write │────────▶│ • Zero-copy  │          │
│  │ • ROCEv2     │  3. Net │ • GPU→GPU    │          │
│  └──────────────┘  transfer└──────────────┘         │
└──────────────────────────────────────────────────────┘
                          │
                          ▼
            ┌──────────────────────────┐
            │ High-Speed Interconnect  │
            │ • InfiniBand (200 Gbps)  │
            │ • RoCE (100-400 Gbps)    │
            │ • NVLink (900 Gbps)      │
            │ • TPU ICI (1.6 Tbps)     │
            └──────────────────────────┘
```

### 3.2 Monitoring and Observability Flow

```
┌────────────────────────────────────────────────────────────┐
│                    vLLM Model Servers                      │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐     │
│  │   Server 1   │  │   Server 2   │  │   Server 3   │     │
│  │              │  │              │  │              │     │
│  │ Metrics:     │  │ Metrics:     │  │ Metrics:     │     │
│  │ • QPS        │  │ • Latency    │  │ • GPU util   │     │
│  │ • TTFT       │  │ • Cache hits │  │ • Memory     │     │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘     │
└─────────┼──────────────────┼──────────────────┼────────────┘
          │                  │                  │
          │ Expose :8000/metrics                │
          └──────────────────┴──────────────────┘
                             │
                             ▼
          ┌─────────────────────────────────────┐
          │      Prometheus (PodMonitor)        │
          │  • Scrapes metrics every 15s        │
          │  • Stores time-series data          │
          │  • Evaluates alerting rules         │
          └─────────────┬───────────────────────┘
                        │
                        ▼
          ┌─────────────────────────────────────┐
          │           Grafana                   │
          │  • Visualize metrics                │
          │  • Custom dashboards                │
          │  • Performance analysis             │
          └─────────────────────────────────────┘
                        │
                        ▼
          ┌─────────────────────────────────────┐
          │       OpenTelemetry (Optional)      │
          │  • Distributed tracing              │
          │  • Request flow tracking            │
          │  • Performance profiling            │
          └─────────────────────────────────────┘
```

---

## 4. Deployment Configuration Flow

```
Developer/Operator
        │
        │ 1. Configure deployment
        ▼
┌────────────────────────────────────────┐
│     guides/<path>/helmfile.yaml        │
│  ┌──────────────────────────────────┐  │
│  │ Defines:                         │  │
│  │ • Namespace                      │  │
│  │ • Release names                  │  │
│  │ • Chart versions                 │  │
│  │ • Values files                   │  │
│  └──────────────────────────────────┘  │
└────────────┬───────────────────────────┘
             │ 2. helmfile apply
             ▼
┌────────────────────────────────────────┐
│        Helmfile Processor              │
│  ┌──────────────────────────────────┐  │
│  │ Renders templates with:          │  │
│  │ • Go templating (.gotmpl)        │  │
│  │ • Environment variables          │  │
│  │ • Values hierarchy               │  │
│  └──────────────────────────────────┘  │
└────────────┬───────────────────────────┘
             │ 3. Generate Helm releases
             ▼
     ┌───────────────────┬───────────────────┐
     │                   │                   │
     ▼                   ▼                   ▼
┌─────────────┐  ┌─────────────┐  ┌─────────────┐
│llm-d-infra  │  │ modelservice│  │ vllm-decode │
│   Chart     │  │   Chart     │  │   Chart     │
│             │  │             │  │             │
│ • Gateway   │  │ • Prefill   │  │ • Decode    │
│ • Scheduler │  │   Workers   │  │   Workers   │
│ • Monitoring│  │ • Config    │  │ • Config    │
└─────┬───────┘  └─────┬───────┘  └─────┬───────┘
      │                │                │
      │ 4. helm install                 │
      └────────────────┴────────────────┘
                       │
                       ▼
          ┌────────────────────────┐
          │  Kubernetes API Server │
          └────────────┬───────────┘
                       │ 5. Create resources
                       ▼
      ┌────────────────────────────────────┐
      │     Kubernetes Resources           │
      │  • Deployments/StatefulSets        │
      │  • Services                        │
      │  • ConfigMaps/Secrets              │
      │  • Gateway/HTTPRoute               │
      │  • InferencePool                   │
      │  • PodMonitors                     │
      └────────────────────────────────────┘
```

---

## 5. Testing and CI/CD Flow

```
Developer
    │
    │ git commit -s
    ▼
┌────────────────────────────────────┐
│      Local Pre-commit Hooks        │
│  • shellcheck                      │
│  • hadolint                        │
│  • markdownlint                    │
│  • yamllint                        │
│  • Custom env var linting          │
└────────────┬───────────────────────┘
             │ git push
             ▼
┌────────────────────────────────────┐
│       GitHub Actions               │
│  ┌──────────────────────────────┐  │
│  │ CI PR Checks                 │  │
│  │ • Lint code                  │  │
│  │ • Validate configs           │  │
│  │ • Check DCO sign-off         │  │
│  └──────────────────────────────┘  │
│                                    │
│  ┌──────────────────────────────┐  │
│  │ Build Images                 │  │
│  │ • Docker build (multi-arch)  │  │
│  │ • Push to ghcr.io            │  │
│  └──────────────────────────────┘  │
│                                    │
│  ┌──────────────────────────────┐  │
│  │ E2E Tests                    │  │
│  │ • inference-scheduling       │  │
│  │ • pd-disaggregation          │  │
│  │ • wide-ep                    │  │
│  │ • Hardware-specific tests    │  │
│  └──────────────────────────────┘  │
└────────────┬───────────────────────┘
             │ On success
             ▼
┌────────────────────────────────────┐
│       Release Process              │
│  • Tag version                     │
│  • Generate changelog              │
│  • Publish container images        │
│  • Update Helm charts              │
└────────────────────────────────────┘
```

---

## Key Takeaways

1. **Layered Architecture**: Clear separation between Gateway, Scheduler, and Model Server layers
2. **Flexible Deployment**: Supports standard, disaggregated, and wide-EP configurations
3. **Intelligent Routing**: Multiple scoring algorithms for optimal backend selection
4. **High-Performance Networking**: RDMA/NIXL for zero-copy KV cache transfer
5. **Cloud-Native**: Built on Kubernetes primitives with Gateway API
6. **Observability**: Comprehensive monitoring with Prometheus and Grafana
7. **Multi-Hardware**: Single codebase supports NVIDIA, AMD, Intel, and Google accelerators
