# NIXL Communication Stack and Plugins

## Overview

**NIXL (Network Inference eXtension Library)** is the communication layer used by llm-d for KV cache transfer in disaggregated serving. NIXL provides an abstraction over high-performance networking for GPU-to-GPU data transfer.

## Underlying Communication Framework: UCX

NIXL is built on top of **UCX (Unified Communication X)**, which provides the actual transport plugins and communication primitives.

### UCX Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    NIXL Library Layer                       │
│           (KV Cache Transfer Abstraction)                   │
└──────────────────────────┬──────────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────────┐
│                  UCX (Unified Communication X)              │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐  │
│  │              UCP (Protocol Layer)                    │  │
│  │  • High-level APIs (send/recv, RMA, AMO)           │  │
│  │  • Connection management                            │  │
│  │  • Protocol selection                               │  │
│  └────────────────────┬─────────────────────────────────┘  │
│                       │                                     │
│  ┌────────────────────▼─────────────────────────────────┐  │
│  │              UCT (Transport Layer)                   │  │
│  │  • Transport-specific implementations               │  │
│  │  • Device access                                    │  │
│  │  • Memory registration                              │  │
│  └────────────────────┬─────────────────────────────────┘  │
│                       │                                     │
│  ┌────────────────────▼─────────────────────────────────┐  │
│  │              UCS (Services Layer)                    │  │
│  │  • Memory management                                │  │
│  │  • Thread management                                │  │
│  │  • Async event handling                             │  │
│  └──────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│              Hardware/Network Transports                    │
│  • InfiniBand Verbs    • RoCE    • EFA (AWS)              │
│  • TCP/IP              • CUDA IPC • Shared Memory          │
└─────────────────────────────────────────────────────────────┘
```

## UCX Transport Plugins (UCX_TLS)

The `UCX_TLS` environment variable controls which transport plugins UCX uses. UCX automatically selects the best available transport based on the configuration.

### Available UCX Transport Plugins

#### 1. **RDMA-based Transports** (Highest Performance)

| Transport | Description | Use Case | Performance |
|-----------|-------------|----------|-------------|
| **rc** | InfiniBand Reliable Connection | InfiniBand networks | Very High (200+ Gbps) |
| **ud** | InfiniBand Unreliable Datagram | InfiniBand multicast | High |
| **dc** | InfiniBand Dynamically Connected | InfiniBand scalable connections | Very High |
| **efa** | AWS Elastic Fabric Adapter | AWS cloud RDMA | Very High (100-400 Gbps) |

#### 2. **GPU-Direct Transports** (GPU Memory Access)

| Transport | Description | Use Case | Performance |
|-----------|-------------|----------|-------------|
| **cuda_ipc** | CUDA Inter-Process Communication | Same-node GPU-to-GPU | Very High (NVLink speed) |
| **cuda_copy** | CUDA memory copy operations | GPU memory operations | High |
| **gdr_copy** | GPUDirect RDMA Copy | GPU→NIC direct transfer | Very High |
| **rocm_ipc** | ROCm Inter-Process Communication | AMD GPU same-node | Very High |

#### 3. **Shared Memory Transports** (Intra-node)

| Transport | Description | Use Case | Performance |
|-----------|-------------|----------|-------------|
| **sm** | Shared Memory | Same-node communication | High |
| **shm** | POSIX Shared Memory | Same-node fallback | Medium |
| **self** | Loopback | Testing/debugging | N/A |

#### 4. **Network Transports** (Fallback/Wide Area)

| Transport | Description | Use Case | Performance |
|-----------|-------------|----------|-------------|
| **tcp** | TCP/IP sockets | Fallback, any network | Medium (10-100 Gbps) |
| **sockcm** | Socket Connection Manager | TCP connection setup | N/A |

## UCX Build Configuration

### Standard CUDA Build (Dockerfile.cuda)
```bash
# UCX is NOT built from source - uses system packages
# Minimal configuration for basic functionality
UCX_TLS environment variable defaults to system UCX capabilities
```

### AWS Build with EFA (Dockerfile.aws)
```bash
# UCX built from source with EFA support
./contrib/configure-release \
    --enable-shared \
    --disable-static \
    --with-cuda=/usr/local/cuda \      # CUDA support
    --with-verbs \                     # InfiniBand verbs
    --with-dm \                        # Device memory
    --with-gdrcopy=/usr/local \        # GPUDirect RDMA
    --with-efa \                       # AWS Elastic Fabric Adapter
    --enable-mt                        # Multi-threading

# Runtime UCX_TLS configuration for AWS:
UCX_TLS="efa,sockcm,sm,self,cuda_copy,cuda_ipc"
```

### GKE/TPU Build (Dockerfile.gke)
```bash
# UCX built from source for RoCE/GKE
./contrib/configure-release \
    --prefix=${UCX_HOME} \
    --with-cuda=${CUDA_HOME} \         # CUDA support
    --with-gdrcopy=${GDRCOPY_HOME} \   # GPUDirect RDMA
    --enable-shared \
    --disable-static \
    --enable-mt \                      # Multi-threading
    --with-verbs \                     # InfiniBand/RoCE verbs
    --with-rdmacm \                    # RDMA connection manager
    --with-dm                          # Device memory
```

## llm-d Deployment Configurations

### 1. Inference Scheduling (Standard)
**Configuration**: Single-node or basic multi-node
```yaml
env:
  - name: UCX_TLS
    value: "cuda_ipc,cuda_copy,tcp"
```

**Transports Used**:
- `cuda_ipc` - GPU-to-GPU on same node (NVLink)
- `cuda_copy` - GPU memory operations
- `tcp` - Fallback for network communication

**Use Case**: Basic deployment without RDMA

### 2. P/D Disaggregation with InfiniBand/RoCE
**Configuration**: Multi-node with RDMA
```yaml
env:
  - name: UCX_TLS
    value: "rc,cuda_ipc,cuda_copy,sm"  # Auto-detected by UCX

resources:
  requests:
    rdma/ib: 1  # InfiniBand RDMA resource
```

**Transports Used**:
- `rc` - InfiniBand Reliable Connection (auto-detected)
- `cuda_ipc` - Same-node GPU communication
- `cuda_copy` - GPU memory operations
- `sm` - Shared memory for same-node

**Use Case**: High-performance P/D disaggregation with KV cache transfer

### 3. AWS with EFA
**Configuration**: AWS cloud with Elastic Fabric Adapter
```yaml
env:
  - name: UCX_TLS
    value: "efa,sockcm,sm,self,cuda_copy,cuda_ipc"

resources:
  requests:
    vpc.amazonaws.com/efa: 1  # AWS EFA resource
```

**Transports Used**:
- `efa` - AWS Elastic Fabric Adapter (RDMA)
- `sockcm` - Socket connection manager
- `sm` - Shared memory
- `self` - Loopback
- `cuda_copy` - GPU memory operations
- `cuda_ipc` - GPU IPC

**Use Case**: AWS deployment with high-performance networking

### 4. Intel XPU
**Configuration**: Intel Data Center GPU Max
```yaml
env:
  - name: UCX_TLS
    value: "tcp,sm,self"  # No CUDA-specific transports
```

**Transports Used**:
- `tcp` - TCP/IP networking
- `sm` - Shared memory
- `self` - Loopback

**Use Case**: Intel XPU deployment

## NIXL Integration with UCX

### How NIXL Uses UCX

```python
# Simplified NIXL workflow
1. Initialize NIXL with UCX backend
   nixl.init(backend="ucx")

2. Register GPU memory buffers
   nixl.register_memory(kv_cache_blocks)

3. Establish connections (uses UCX_TLS transports)
   conn = nixl.connect(target_host, target_port)

4. Transfer KV cache (zero-copy if GPUDirect available)
   nixl.send(conn, kv_cache_data)
   nixl.recv(conn, kv_cache_buffer)

5. Cleanup
   nixl.close(conn)
```

### NIXL Connector in vLLM

In vLLM's P/D disaggregation configuration:
```yaml
args:
  - "--kv-transfer-config"
  - '{"kv_connector":"NixlConnector", "kv_role":"kv_both"}'

env:
  - name: VLLM_NIXL_SIDE_CHANNEL_HOST
    valueFrom:
      fieldRef:
        fieldPath: status.podIP
```

## Transport Selection Priority

UCX automatically selects the best available transport in this order:

1. **GPU-Direct RDMA** (if available)
   - `rc` + `gdr_copy` (InfiniBand)
   - `efa` + `cuda_copy` (AWS)

2. **RDMA** (if available)
   - `rc` (InfiniBand)
   - `efa` (AWS)

3. **GPU IPC** (same node only)
   - `cuda_ipc` (NVIDIA)
   - `rocm_ipc` (AMD)

4. **Shared Memory** (same node)
   - `sm`, `shm`

5. **TCP/IP** (fallback)
   - `tcp`

## Performance Characteristics

| Transport | Latency | Bandwidth | Zero-Copy GPU | Use Case |
|-----------|---------|-----------|---------------|----------|
| **cuda_ipc** | ~1 μs | 900 Gbps (NVLink) | Yes | Same-node GPUs |
| **rc (IB)** | ~1-2 μs | 200-400 Gbps | With GPUDirect | Cross-node RDMA |
| **efa** | ~2-5 μs | 100-400 Gbps | With GPUDirect | AWS cross-node |
| **tcp** | ~10-100 μs | 10-100 Gbps | No | Fallback |

## Key Configuration Variables

### UCX Environment Variables

```bash
# Transport selection
UCX_TLS="rc,cuda_ipc,cuda_copy"        # Specify transports

# Memory hooks (required for CUDA)
UCX_MEM_MMAP_HOOK_MODE=none            # Disable mmap hooks for CUDA

# Network interface selection
UCX_NET_DEVICES=mlx5_0:1              # Specific InfiniBand device

# RDMA device selection
UCX_IB_GPU_DIRECT_RDMA=yes            # Enable GPUDirect RDMA

# Debug and logging
UCX_LOG_LEVEL=info                    # UCX logging level
```

### NIXL Environment Variables

```bash
# NIXL configuration
VLLM_NIXL_SIDE_CHANNEL_HOST=<pod_ip>  # Control plane IP
VLLM_NIXL_SIDE_CHANNEL_PORT=8300      # Control plane port
```

## Troubleshooting

### Check Available UCX Transports
```bash
ucx_info -d
```

### Test UCX Performance
```bash
ucx_perftest -t tag_lat -s 1048576  # Latency test
ucx_perftest -t tag_bw -s 1048576   # Bandwidth test
```

### Verify RDMA Device
```bash
ibv_devices                          # List IB devices
rdma_lat                             # RDMA latency test
```

### Check NIXL Connection
```bash
# In vLLM logs
grep "NIXL" /var/log/vllm.log
```

## Summary

**NIXL uses UCX as its communication plugin framework**, with UCX providing:

1. **Transport abstraction** via `UCX_TLS` plugins
2. **Multiple high-performance transports**:
   - RDMA (InfiniBand, RoCE, EFA)
   - GPU-Direct (CUDA IPC, GPUDirect RDMA)
   - Shared Memory
   - TCP/IP fallback
3. **Automatic transport selection** based on hardware capabilities
4. **Zero-copy GPU memory transfer** when GPUDirect is available

The specific transports used depend on the deployment environment and hardware configuration, with UCX automatically selecting the optimal transport combination.
