# Complete Intel BMG Deployment Guide for llm-d

This comprehensive guide covers all aspects of deploying llm-d on Intel BMG (Battlemage G21) GPUs, including UCX/NIXL configuration.

## Quick Navigation

- **New to Intel BMG?** → Start with [Setup Guide](./intel-bmg-setup.md)
- **Need quick commands?** → See [Quick Reference](./intel-bmg-quick-reference.md)
- **UCX/NIXL questions?** → Read [UCX Configuration](#ucx-and-nixl-for-intel-bmg) below

## Documentation Structure

### 1. [Intel BMG Setup Guide](./intel-bmg-setup.md)
**Complete deployment walkthrough**
- Prerequisites and installation
- Step-by-step deployment
- Verification and testing
- Troubleshooting

### 2. [Intel BMG Quick Reference](./intel-bmg-quick-reference.md)
**One-page cheat sheet**
- Key configuration changes
- Quick deploy commands
- Common issues table

### 3. [UCX Configuration Guide](./ucx-configuration-guide.md)
**Deep dive into UCX**
- UCX architecture and transports
- Hardware-specific configurations
- Performance tuning
- Advanced troubleshooting

## Intel BMG Overview

### What Makes BMG Different?

| Aspect | Intel Data Center GPU Max 1550 | Intel BMG (Battlemage G21) |
|--------|-------------------------------|---------------------------|
| GPU Resource | `gpu.intel.com/i915` | `gpu.intel.com/xe` |
| Network Support | Can use RDMA with proper setup | TCP only |
| Use Case | Data center / HPC | Development / smaller deployments |
| Container Image | Same (Dockerfile.xpu) | Same |

### When to Use Intel BMG

**Good For:**
- Development and testing
- Small-scale inference deployments
- Cost-effective GPU inference
- Inference scheduling workloads

**Not Ideal For:**
- Large-scale P/D disaggregation requiring RDMA
- Wide Expert Parallelism requiring high-speed interconnects
- Workloads requiring > 100 Gbps network bandwidth

## UCX and NIXL for Intel BMG

### What is UCX?

**UCX (Unified Communication X)** is a communication framework that NIXL uses for data transfer between inference instances.

```
vLLM (Inference Engine)
    ↓
NIXL (KV Cache Transfer)
    ↓
UCX (Communication Layer) ← You configure this
    ↓
Network (TCP/RDMA/InfiniBand)
```

### Do I Need to Configure UCX?

| Deployment Type | UCX Needed? | Configuration |
|----------------|-------------|---------------|
| Inference Scheduling | ❌ No | N/A |
| Simple single-node | ❌ No | N/A |
| P/D Disaggregation | ✅ Yes | Set `UCX_TLS=tcp` |
| Wide Expert Parallelism | ✅ Yes | Set `UCX_TLS=tcp` |

### UCX Configuration for BMG

**Key Environment Variable:**

```yaml
env:
  - name: UCX_TLS
    value: "tcp"
```

**What UCX_TLS Does:**
- `UCX_TLS` = "Transport Layer Selection"
- Tells UCX which network protocols to use
- For BMG: Only `tcp` is supported

**Why TCP for BMG?**
1. BMG lacks specialized interconnects (no InfiniBand/RoCE)
2. TCP works over standard Ethernet
3. Sufficient for BMG's typical workloads

### Complete UCX Configuration (P/D Disaggregation)

```yaml
routing:
  proxy:
    connector: nixlv2  # NIXL version 2
    secure: false

decode:
  containers:
  - name: "vllm"
    args:
      - "--kv-transfer-config"
      - '{"kv_connector":"NixlConnector", "kv_role":"kv_both", "kv_buffer_device":"cpu"}'

    env:
      # 1. UCX Transport Configuration
      - name: UCX_TLS
        value: "tcp"

      # 2. NIXL Side Channel (for control plane)
      - name: VLLM_NIXL_SIDE_CHANNEL_HOST
        valueFrom:
          fieldRef:
            fieldPath: status.podIP
      - name: VLLM_NIXL_SIDE_CHANNEL_PORT
        value: "5557"

      # 3. Intel XPU Configuration
      - name: VLLM_WORKER_MULTIPROC_METHOD
        value: "spawn"
      - name: VLLM_TARGET_DEVICE
        value: "xpu"
```

## Complete Deployment Examples

### Example 1: Inference Scheduling (No UCX Needed)

```bash
# Setup
export NAMESPACE=llm-d-bmg
kubectl create namespace ${NAMESPACE}
kubectl create secret generic llm-d-hf-token \
  --from-literal="HF_TOKEN=" \
  --namespace ${NAMESPACE}

# Deploy gateway
cd guides/prereq/gateway-provider
./install-gateway-provider-dependencies.sh
helmfile apply -f istio.helmfile.yaml

# Deploy inference scheduling
cd ../../inference-scheduling

# Option A: Use BMG-specific values
helmfile apply -n ${NAMESPACE} \
  --state-values-set modelservice.valuesFiles='{ms-inference-scheduling/values_bmg.yaml}'

# Option B: Use XPU values and override GPU resource
helmfile apply -e xpu -n ${NAMESPACE}
# Then edit deployment to change i915 → xe

# Install HTTPRoute
kubectl apply -f httproute.yaml -n ${NAMESPACE}
```

**Key Points:**
- No UCX configuration needed
- GPU resource must be `gpu.intel.com/xe`
- Works with standard networking

### Example 2: P/D Disaggregation (UCX Required)

```bash
# Setup
export NAMESPACE=llm-d-pd-bmg
export HF_TOKEN=your-token

kubectl create namespace ${NAMESPACE}
kubectl create secret generic llm-d-hf-token \
  --from-literal="HF_TOKEN=${HF_TOKEN}" \
  --namespace ${NAMESPACE}

# Deploy gateway (same as above)
cd guides/prereq/gateway-provider
./install-gateway-provider-dependencies.sh
helmfile apply -f istio.helmfile.yaml

# Deploy P/D with BMG values
cd ../../pd-disaggregation

# Use BMG-specific values file
helmfile apply -n ${NAMESPACE} \
  --state-values-set modelservice.valuesFiles='{ms-pd/values_bmg.yaml}'

# Install HTTPRoute
kubectl apply -f httproute.yaml -n ${NAMESPACE}

# Verify UCX configuration
kubectl get pod -n ${NAMESPACE} -l llm-d.ai/role=decode -o yaml | grep UCX_TLS
# Should show: value: "tcp"
```

**Key Points:**
- Must set `UCX_TLS=tcp`
- NIXL side channel configured
- KV cache transfer over TCP
- Total: 4 BMG GPUs (1 decode + 3 prefill)

## Verification and Testing

### 1. Check GPU Resources

```bash
# Verify GPU plugin detects BMG
kubectl get nodes -o json | \
  jq '.items[].status.capacity | select(.["gpu.intel.com/xe"] != null)'

# Should show gpu.intel.com/xe resources
```

### 2. Verify UCX Configuration (P/D only)

```bash
# Check decode pod
DECODE_POD=$(kubectl get pods -n ${NAMESPACE} \
  -l llm-d.ai/role=decode -o jsonpath='{.items[0].metadata.name}')

# Check UCX_TLS
kubectl exec -n ${NAMESPACE} ${DECODE_POD} -c vllm -- \
  printenv UCX_TLS
# Should show: tcp

# Check NIXL side channel
kubectl exec -n ${NAMESPACE} ${DECODE_POD} -c vllm -- \
  printenv | grep VLLM_NIXL
# Should show HOST and PORT variables
```

### 3. Test KV Cache Transfer (P/D only)

```bash
# Port forward to gateway
kubectl port-forward -n ${NAMESPACE} \
  service/infra-*-inference-gateway-istio 8086:80 &

# Send inference request
curl -X POST "http://localhost:8086/v1/chat/completions" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "deepseek-ai/DeepSeek-R1-Distill-Qwen-1.5B",
    "messages": [{"role": "user", "content": "Test P/D disaggregation"}],
    "max_tokens": 100
  }'

# Check decode logs for KV cache transfer
kubectl logs -n ${NAMESPACE} ${DECODE_POD} -c vllm | \
  grep -i "kv.*transfer\|nixl"
```

## Troubleshooting

### Issue: Wrong GPU Resource Type

**Symptom**: Pods stuck in Pending

```bash
kubectl describe pod <pod-name> -n ${NAMESPACE}
# Shows: 0/1 nodes are available: Insufficient gpu.intel.com/xe
```

**Solution**: Check you're using `xe` not `i915`

```bash
kubectl get pod <pod-name> -n ${NAMESPACE} -o yaml | \
  grep "gpu.intel.com"
# Should show: gpu.intel.com/xe: "1"
```

### Issue: UCX Transport Not Available

**Symptom**: NIXL connection failures

```bash
kubectl logs <pod-name> -n ${NAMESPACE} | grep -i "ucx.*error"
# Shows: UCX ERROR: No transport available
```

**Solution**: Verify TCP transport is specified

```bash
# Check UCX_TLS value
kubectl get pod <pod-name> -n ${NAMESPACE} -o yaml | \
  grep -A 1 "UCX_TLS"

# Should show:
#   name: UCX_TLS
#   value: tcp
```

### Issue: NIXL Side Channel Errors

**Symptom**: Can't connect to side channel

```bash
kubectl logs <pod-name> -n ${NAMESPACE} | grep -i "side channel"
# Shows: Failed to connect to side channel
```

**Solution**: Verify side channel configuration

```bash
# Check variables are set
kubectl exec <pod-name> -n ${NAMESPACE} -c vllm -- printenv | \
  grep VLLM_NIXL_SIDE_CHANNEL

# Should show:
#   VLLM_NIXL_SIDE_CHANNEL_HOST=<pod-ip>
#   VLLM_NIXL_SIDE_CHANNEL_PORT=5557

# Verify port is listening
kubectl exec <pod-name> -n ${NAMESPACE} -c vllm -- \
  netstat -ln | grep 5557
```

## Performance Expectations

### TCP vs RDMA Performance

| Metric | TCP (BMG) | RDMA (InfiniBand) |
|--------|-----------|-------------------|
| Bandwidth | 10-100 Gbps | 100-400 Gbps |
| Latency | 10-100 µs | 1-10 µs |
| KV Transfer | Adequate for small models | Best for large models |

**BMG P/D Performance Tips:**
1. Use smaller models (< 10B parameters)
2. Keep input/output sequences reasonable (< 32K tokens)
3. Adjust prefill/decode ratios based on workload
4. Monitor network utilization

### Expected Throughput

**Inference Scheduling (2 replicas):**
- Small models (0.6B-1.5B): 50-100 req/s
- Medium models (7B-8B): 10-20 req/s

**P/D Disaggregation (1 decode + 3 prefill):**
- With TCP: 70-80% of RDMA performance
- Long sequences may see higher latency
- Best for batch processing

## Next Steps

### After Successful Deployment

1. **Monitor Performance**
   ```bash
   # View metrics
   kubectl port-forward -n ${NAMESPACE} service/gaie-*-epp 9090:9090
   # Access Prometheus at http://localhost:9090
   ```

2. **Scale Deployment**
   ```bash
   # Increase replicas
   kubectl scale deployment <deployment-name> -n ${NAMESPACE} --replicas=4
   ```

3. **Try Different Models**
   - Edit `modelArtifacts.uri` in values file
   - Redeploy with `helmfile apply`

### Learning More

- **UCX Deep Dive**: [docs/ucx-configuration-guide.md](./ucx-configuration-guide.md)
- **P/D Architecture**: [guides/pd-disaggregation/README.md](../guides/pd-disaggregation/README.md)
- **Inference Scheduling**: [guides/inference-scheduling/README.md](../guides/inference-scheduling/README.md)
- **llm-d Documentation**: https://www.llm-d.ai

## Quick Command Reference

```bash
# Check GPU resources
kubectl get nodes -o json | jq '.items[].status.capacity | select(.["gpu.intel.com/xe"] != null)'

# Deploy inference scheduling
helmfile apply -n llm-d-bmg \
  --state-values-set modelservice.valuesFiles='{ms-inference-scheduling/values_bmg.yaml}'

# Deploy P/D disaggregation
helmfile apply -n llm-d-pd-bmg \
  --state-values-set modelservice.valuesFiles='{ms-pd/values_bmg.yaml}'

# Check UCX configuration
kubectl get pod <pod> -n <ns> -o yaml | grep UCX_TLS

# View NIXL logs
kubectl logs <pod> -n <ns> -c vllm | grep -i nixl

# Test inference
kubectl port-forward -n <ns> service/infra-*-inference-gateway-istio 8086:80
curl http://localhost:8086/health
```

## Support and Resources

- **llm-d Slack**: https://llm-d.ai/slack
- **GitHub Issues**: https://github.com/llm-d/llm-d/issues
- **Intel XPU Maintainer**: Yuan Wu (yuan.wu@intel.com)
- **UCX Documentation**: https://openucx.readthedocs.io/

## Related Documentation

- [Intel BMG Setup Guide](./intel-bmg-setup.md) - Detailed setup instructions
- [Intel BMG Quick Reference](./intel-bmg-quick-reference.md) - One-page cheat sheet
- [UCX Configuration Guide](./ucx-configuration-guide.md) - Complete UCX reference
- [Intel Data Center GPU Max Guide](./accelerators/README.md) - For i915-based GPUs
