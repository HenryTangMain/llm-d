# UCX Configuration Guide for llm-d

This guide explains how to configure UCX (Unified Communication X) for different hardware and networking scenarios in llm-d deployments.

## Table of Contents

- [What is UCX?](#what-is-ucx)
- [UCX in llm-d Architecture](#ucx-in-llm-d-architecture)
- [UCX Transport Layers (UCX_TLS)](#ucx-transport-layers-ucx_tls)
- [Configuration by Hardware](#configuration-by-hardware)
- [Advanced UCX Configuration](#advanced-ucx-configuration)
- [Troubleshooting](#troubleshooting)
- [Performance Tuning](#performance-tuning)

## What is UCX?

**UCX (Unified Communication X)** is a high-performance communication framework that provides:

- **Multiple Transport Layers**: TCP, RDMA (InfiniBand/RoCE), shared memory, CUDA IPC
- **Optimized Data Transfer**: Zero-copy transfers, RDMA operations
- **Hardware Acceleration**: Support for GPUDirect RDMA, EFA, InfiniBand
- **Flexible APIs**: Used by NIXL for KV cache transfer in P/D disaggregation

## UCX in llm-d Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    llm-d Stack                          │
├─────────────────────────────────────────────────────────┤
│  vLLM (Inference Engine)                                │
│    ├─ KV Cache Transfer ──────────────┐                │
│    └─ Expert Parallelism ──────────┐  │                │
├─────────────────────────────────────┼──┼────────────────┤
│  NIXL (KV Cache Communication)      │  │                │
│    └─ Uses UCX for transport ───────┘  │                │
├─────────────────────────────────────────┼────────────────┤
│  UCX (Communication Framework)          │                │
│    ├─ Transport Selection (UCX_TLS)    │                │
│    ├─ RDMA / TCP / Shared Memory       │                │
│    └─ Hardware Acceleration ────────────┘                │
├─────────────────────────────────────────────────────────┤
│  Hardware Layer                                          │
│    ├─ InfiniBand / RoCE                                 │
│    ├─ AWS EFA                                            │
│    ├─ Standard TCP/IP                                   │
│    └─ GPU Direct RDMA                                    │
└─────────────────────────────────────────────────────────┘
```

### When UCX is Used

UCX is essential for:

1. **P/D Disaggregation**: KV cache transfer between prefill and decode instances
2. **Wide Expert Parallelism**: Communication between MoE model instances
3. **Multi-node Inference**: Any deployment spanning multiple Kubernetes nodes

UCX is **NOT needed** for:
- Single-node inference without P/D disaggregation
- Simple inference scheduling without KV cache transfer

## UCX Transport Layers (UCX_TLS)

The `UCX_TLS` environment variable controls which transport layers UCX uses. UCX automatically selects the best available transport, but you can override this.

### Transport Layer Options

| Transport | Description | Use Case | Performance |
|-----------|-------------|----------|-------------|
| `tcp` | Standard TCP/IP networking | Intel BMG, standard networks | Low (50-100 Gbps) |
| `rc` | RDMA Connection-oriented | InfiniBand clusters | High (100-400 Gbps) |
| `ud` | RDMA Unreliable Datagram | InfiniBand for small messages | High |
| `efa` | AWS Elastic Fabric Adapter | AWS EC2 with EFA | High (100-400 Gbps) |
| `sm` | Shared memory | Same-node communication | Highest |
| `cuda_copy` | CUDA memory copy | GPU-GPU transfers | High |
| `cuda_ipc` | CUDA IPC | Same-node GPU-GPU | Highest |
| `self` | Loopback | Local testing | N/A |
| `sockcm` | Socket connection manager | Connection setup | N/A |

### Common UCX_TLS Configurations

```yaml
# Intel BMG - TCP only
env:
  - name: UCX_TLS
    value: "tcp"

# AWS EFA - Multi-transport with EFA priority
env:
  - name: UCX_TLS
    value: "efa,sockcm,sm,self,cuda_copy,cuda_ipc"

# InfiniBand/RoCE - RDMA with fallback
env:
  - name: UCX_TLS
    value: "rc,ud,sm,self,cuda_copy,cuda_ipc"

# TCP with CUDA acceleration
env:
  - name: UCX_TLS
    value: "tcp,sm,self,cuda_copy,cuda_ipc"
```

## Configuration by Hardware

### Intel BMG (Battlemage G21)

**Network**: Standard TCP/IP
**UCX Configuration**: TCP only

```yaml
decode:
  containers:
  - name: "vllm"
    image: ghcr.io/llm-d/llm-d-xpu:v0.3.1
    env:
      # UCX Transport: TCP only
      - name: UCX_TLS
        value: "tcp"

      # NIXL side channel
      - name: VLLM_NIXL_SIDE_CHANNEL_HOST
        valueFrom:
          fieldRef:
            fieldPath: status.podIP
      - name: VLLM_NIXL_SIDE_CHANNEL_PORT
        value: "5557"

    resources:
      limits:
        gpu.intel.com/xe: 1
      requests:
        gpu.intel.com/xe: 1

prefill:
  containers:
  - name: "vllm"
    image: ghcr.io/llm-d/llm-d-xpu:v0.3.1
    env:
      - name: UCX_TLS
        value: "tcp"
      - name: VLLM_NIXL_SIDE_CHANNEL_HOST
        valueFrom:
          fieldRef:
            fieldPath: status.podIP
      - name: VLLM_NIXL_SIDE_CHANNEL_PORT
        value: "5557"
    resources:
      limits:
        gpu.intel.com/xe: 1
```

**Why TCP for BMG?**
- Intel BMG lacks specialized high-speed interconnects
- No InfiniBand or RoCE support
- TCP is reliable and universally available

### NVIDIA GPUs with InfiniBand

**Network**: InfiniBand (100-400 Gbps)
**UCX Configuration**: RDMA with CUDA acceleration

```yaml
decode:
  containers:
  - name: "vllm"
    image: ghcr.io/llm-d/llm-d-cuda:v0.3.1
    env:
      # UCX Transport: RDMA (rc/ud) with CUDA support
      - name: UCX_TLS
        value: "rc,ud,sm,self,cuda_copy,cuda_ipc"

      # Optional: Specify IB device
      - name: UCX_NET_DEVICES
        value: "mlx5_0:1"  # InfiniBand device

      # NIXL side channel
      - name: VLLM_NIXL_SIDE_CHANNEL_HOST
        valueFrom:
          fieldRef:
            fieldPath: status.podIP
      - name: VLLM_NIXL_SIDE_CHANNEL_PORT
        value: "5557"

    resources:
      limits:
        nvidia.com/gpu: "4"
        rdma/ib: 1  # Request InfiniBand resource
      requests:
        nvidia.com/gpu: "4"
        rdma/ib: 1

prefill:
  containers:
  - name: "vllm"
    image: ghcr.io/llm-d/llm-d-cuda:v0.3.1
    env:
      - name: UCX_TLS
        value: "rc,ud,sm,self,cuda_copy,cuda_ipc"
      - name: UCX_NET_DEVICES
        value: "mlx5_0:1"
      - name: VLLM_NIXL_SIDE_CHANNEL_HOST
        valueFrom:
          fieldRef:
            fieldPath: status.podIP
    resources:
      limits:
        nvidia.com/gpu: "1"
        rdma/ib: 1
      requests:
        nvidia.com/gpu: "1"
        rdma/ib: 1
```

**Key Points:**
- Request `rdma/ib` resource in Kubernetes
- UCX will automatically use RDMA for high-bandwidth transfers
- Shared memory (`sm`) used for same-node communication

### NVIDIA GPUs on GKE with RoCE

**Network**: RoCE (RDMA over Converged Ethernet)
**UCX Configuration**: Similar to InfiniBand

```yaml
decode:
  containers:
  - name: "vllm"
    image: ghcr.io/llm-d/llm-d-cuda:v0.3.1
    env:
      # RoCE uses similar transports to InfiniBand
      - name: UCX_TLS
        value: "rc,ud,sm,self,cuda_copy,cuda_ipc"

      - name: VLLM_NIXL_SIDE_CHANNEL_HOST
        valueFrom:
          fieldRef:
            fieldPath: status.podIP

    resources:
      limits:
        nvidia.com/gpu: "4"
        # GKE may use different resource names for RDMA
        # Check your cluster's available resources
      requests:
        nvidia.com/gpu: "4"
```

**GKE-Specific Notes:**
- RoCE provides RDMA over standard Ethernet
- Validated on 8xH200 clusters
- Check GKE documentation for RDMA resource names

### AWS EC2 with EFA (Elastic Fabric Adapter)

**Network**: AWS EFA (up to 400 Gbps)
**UCX Configuration**: EFA-optimized transport

```yaml
decode:
  containers:
  - name: "vllm"
    image: ghcr.io/llm-d/llm-d-cuda-efa:v0.3.1  # EFA-enabled image
    env:
      # EFA transport with fallbacks
      - name: UCX_TLS
        value: "efa,sockcm,sm,self,cuda_copy,cuda_ipc"

      # Optional: EFA-specific tuning
      - name: FI_EFA_USE_DEVICE_RDMA
        value: "1"

      - name: VLLM_NIXL_SIDE_CHANNEL_HOST
        valueFrom:
          fieldRef:
            fieldPath: status.podIP

    resources:
      limits:
        nvidia.com/gpu: "4"
        vpc.amazonaws.com/efa: 1  # Request EFA resource
      requests:
        nvidia.com/gpu: "4"
        vpc.amazonaws.com/efa: 1

prefill:
  containers:
  - name: "vllm"
    image: ghcr.io/llm-d/llm-d-cuda-efa:v0.3.1
    env:
      - name: UCX_TLS
        value: "efa,sockcm,sm,self,cuda_copy,cuda_ipc"
      - name: FI_EFA_USE_DEVICE_RDMA
        value: "1"
      - name: VLLM_NIXL_SIDE_CHANNEL_HOST
        valueFrom:
          fieldRef:
            fieldPath: status.podIP
    resources:
      limits:
        nvidia.com/gpu: "1"
        vpc.amazonaws.com/efa: 1
      requests:
        nvidia.com/gpu: "1"
        vpc.amazonaws.com/efa: 1
```

**EFA Requirements:**
- Use EFA-enabled container image (`Dockerfile.aws`)
- Request `vpc.amazonaws.com/efa` resource
- EC2 instance types: p4d, p5, etc.

### Intel Data Center GPU Max 1550

**Network**: Varies by deployment
**UCX Configuration**: TCP or RDMA depending on network

```yaml
# TCP configuration (similar to BMG)
decode:
  containers:
  - name: "vllm"
    image: ghcr.io/llm-d/llm-d-xpu:v0.3.1
    env:
      - name: UCX_TLS
        value: "tcp"
      - name: VLLM_NIXL_SIDE_CHANNEL_HOST
        valueFrom:
          fieldRef:
            fieldPath: status.podIP
    resources:
      limits:
        gpu.intel.com/i915: 1  # i915 for Data Center GPU Max
      requests:
        gpu.intel.com/i915: 1
```

## Advanced UCX Configuration

### Additional UCX Environment Variables

```yaml
env:
  # Transport layer selection
  - name: UCX_TLS
    value: "rc,ud,sm,self,cuda_copy,cuda_ipc"

  # Network device selection
  - name: UCX_NET_DEVICES
    value: "mlx5_0:1"  # Specific InfiniBand device

  # Memory type cache (for GPU memory)
  - name: UCX_MEMTYPE_CACHE
    value: "n"  # Disable if causing issues

  # Rendezvous threshold (bytes)
  - name: UCX_RNDV_THRESH
    value: "8192"  # Use RDMA for messages > 8KB

  # InfiniBand settings
  - name: UCX_IB_GPU_DIRECT_RDMA
    value: "yes"  # Enable GPUDirect RDMA

  # Disable memory hooks (if conflicts with allocators)
  - name: UCX_MEM_MMAP_HOOK_MODE
    value: "none"

  # Debug logging
  - name: UCX_LOG_LEVEL
    value: "info"  # Options: error, warn, info, debug, trace
```

### Network Device Selection

To specify which network interface UCX should use:

```yaml
env:
  # Automatic selection by device name pattern
  - name: UCX_NET_DEVICES
    value: "mlx5_0:1,mlx5_1:1"  # Multiple IB devices

  # Or by Ethernet interface
  - name: UCX_NET_DEVICES
    value: "eth0"  # Standard Ethernet
```

Find your devices:
```bash
# List InfiniBand devices
ibstat

# List network interfaces
ip link show

# Check UCX device detection
ucx_info -d
```

### GPUDirect RDMA Configuration

Enable direct GPU-to-GPU transfers over RDMA:

```yaml
env:
  - name: UCX_IB_GPU_DIRECT_RDMA
    value: "yes"

  # Ensure CUDA IPC is in transport list
  - name: UCX_TLS
    value: "rc,ud,sm,self,cuda_copy,cuda_ipc"
```

**Requirements:**
- NVIDIA GPUDirect RDMA kernel module
- InfiniBand or RoCE network
- Mellanox/NVIDIA NICs

### Memory Management

UCX memory management settings:

```yaml
env:
  # Disable memory hooks if using custom allocators
  - name: UCX_MEM_MMAP_HOOK_MODE
    value: "none"

  # Memory type cache for GPU memory
  - name: UCX_MEMTYPE_CACHE
    value: "y"  # Enable caching (default)
```

## Troubleshooting

### Problem: NIXL Connection Failures

**Symptoms:**
```
ERROR: Failed to connect to peer
UCX transport not available
```

**Diagnosis:**
```bash
# Check UCX can see network devices
kubectl exec -it <pod-name> -n <namespace> -- ucx_info -d

# Check UCX_TLS value
kubectl get pod <pod-name> -n <namespace> -o yaml | grep UCX_TLS

# View NIXL logs
kubectl logs <pod-name> -n <namespace> -c vllm | grep -i nixl
```

**Solutions:**

1. **Wrong transport specified:**
   ```yaml
   # Intel BMG - should use tcp, not rdma
   - name: UCX_TLS
     value: "tcp"  # Not "rc" or "efa"
   ```

2. **Missing RDMA resources:**
   ```yaml
   # Request RDMA device for InfiniBand
   resources:
     limits:
       rdma/ib: 1
   ```

3. **Network device not found:**
   ```yaml
   # Remove or fix UCX_NET_DEVICES
   - name: UCX_NET_DEVICES
     value: "mlx5_0:1"  # Ensure device exists
   ```

### Problem: Poor KV Cache Transfer Performance

**Symptoms:**
- High P/D latency
- Low throughput in disaggregated setup

**Diagnosis:**
```bash
# Check actual transport being used
kubectl logs <pod-name> -n <namespace> | grep "UCX.*using"

# Monitor network usage
kubectl exec -it <pod-name> -- iftop -i eth0
```

**Solutions:**

1. **Enable RDMA if available:**
   ```yaml
   # Switch from TCP to RDMA
   - name: UCX_TLS
     value: "rc,ud,sm,self,cuda_copy,cuda_ipc"
   ```

2. **Tune rendezvous threshold:**
   ```yaml
   # Use RDMA for smaller messages
   - name: UCX_RNDV_THRESH
     value: "4096"  # Lower threshold
   ```

3. **Enable GPUDirect RDMA:**
   ```yaml
   - name: UCX_IB_GPU_DIRECT_RDMA
     value: "yes"
   ```

### Problem: UCX Initialization Errors

**Symptoms:**
```
UCX ERROR: No transport available
Failed to create UCX context
```

**Solutions:**

1. **Add TCP as fallback:**
   ```yaml
   - name: UCX_TLS
     value: "rc,ud,tcp,sm,self,cuda_copy,cuda_ipc"
   ```

2. **Check UCX is built in container:**
   ```bash
   kubectl exec -it <pod-name> -- ucx_info -v
   ```

3. **Verify network devices:**
   ```bash
   kubectl exec -it <pod-name> -- ip link show
   kubectl exec -it <pod-name> -- ibstat  # For InfiniBand
   ```

### Problem: Memory Allocation Failures

**Symptoms:**
```
UCX ERROR: failed to allocate memory
```

**Solutions:**

1. **Disable memory hooks:**
   ```yaml
   - name: UCX_MEM_MMAP_HOOK_MODE
     value: "none"
   ```

2. **Increase shared memory:**
   ```yaml
   volumes:
     - name: dshm
       emptyDir:
         medium: Memory
         sizeLimit: 8Gi  # Increase from 4Gi
   ```

## Performance Tuning

### Transport Selection Priority

UCX tries transports in the order specified:

```yaml
# High-performance order (InfiniBand)
- name: UCX_TLS
  value: "rc,ud,sm,self,cuda_copy,cuda_ipc"
  # 1. Try RDMA RC first (best for large transfers)
  # 2. Fall back to UD (better for small messages)
  # 3. Use shared memory for same-node
  # 4. CUDA acceleration for GPU-GPU

# AWS EFA order
- name: UCX_TLS
  value: "efa,sockcm,sm,self,cuda_copy,cuda_ipc"
  # 1. Prioritize EFA
  # 2. Socket connection manager
  # 3. Shared memory and CUDA
```

### Bandwidth Optimization

For maximum bandwidth:

```yaml
env:
  # Enable RDMA
  - name: UCX_TLS
    value: "rc,sm,self,cuda_copy,cuda_ipc"

  # Lower rendezvous threshold
  - name: UCX_RNDV_THRESH
    value: "8192"

  # Enable GPUDirect RDMA
  - name: UCX_IB_GPU_DIRECT_RDMA
    value: "yes"

  # Use all available IB ports
  - name: UCX_NET_DEVICES
    value: "mlx5_0:1,mlx5_1:1"
```

### Latency Optimization

For minimum latency:

```yaml
env:
  # Prefer unreliable datagram for low latency
  - name: UCX_TLS
    value: "ud,rc,sm,self,cuda_copy,cuda_ipc"

  # Higher threshold to avoid RDMA overhead for small messages
  - name: UCX_RNDV_THRESH
    value: "16384"
```

### Debugging and Monitoring

Enable UCX logging:

```yaml
env:
  # Detailed UCX logging
  - name: UCX_LOG_LEVEL
    value: "debug"

  # Log file location
  - name: UCX_LOG_FILE
    value: "/tmp/ucx.log"

  # Print configuration on startup
  - name: UCX_LOG_PRINT_CONFIG
    value: "y"
```

View logs:
```bash
kubectl logs <pod-name> -n <namespace> | grep UCX
kubectl exec -it <pod-name> -- cat /tmp/ucx.log
```

## Summary: Quick Configuration Matrix

| Hardware | Network | UCX_TLS Value | Additional Resources |
|----------|---------|---------------|---------------------|
| Intel BMG | TCP/IP | `tcp` | None |
| Intel Max 1550 | TCP/IP | `tcp` | None |
| NVIDIA + InfiniBand | IB | `rc,ud,sm,self,cuda_copy,cuda_ipc` | `rdma/ib: 1` |
| NVIDIA + RoCE | Ethernet | `rc,ud,sm,self,cuda_copy,cuda_ipc` | Provider-specific |
| NVIDIA + AWS EFA | EFA | `efa,sockcm,sm,self,cuda_copy,cuda_ipc` | `vpc.amazonaws.com/efa: 1` |
| Any + TCP only | TCP/IP | `tcp,sm,self,cuda_copy,cuda_ipc` | None |

## Additional Resources

- **UCX Documentation**: https://openucx.readthedocs.io/
- **UCX GitHub**: https://github.com/openucx/ucx
- **NIXL Integration**: Built into llm-d container images
- **llm-d P/D Guide**: [guides/pd-disaggregation/README.md](../guides/pd-disaggregation/README.md)
- **Container Images**:
  - CUDA: `docker/Dockerfile.cuda`
  - AWS EFA: `docker/Dockerfile.aws`
  - Intel XPU: `docker/Dockerfile.xpu`
  - GKE: `docker/Dockerfile.gke`

## Support

For UCX-specific issues:
- Check UCX GitHub issues
- Review UCX documentation
- Contact hardware vendor support

For llm-d integration:
- **Slack**: https://llm-d.ai/slack
- **GitHub Issues**: https://github.com/llm-d/llm-d/issues
