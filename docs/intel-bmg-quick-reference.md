# Intel BMG Quick Reference

Fast reference guide for deploying llm-d on Intel BMG (Battlemage G21) GPUs.

## Prerequisites Checklist

```bash
# 1. Install Intel GPU Device Plugin
kubectl apply -k 'https://github.com/intel/intel-device-plugins-for-kubernetes/deployments/gpu_plugin?ref=v0.32.1'

# 2. Verify GPU resources
kubectl get nodes -o json | jq '.items[].status.capacity | select(.["gpu.intel.com/xe"] != null)'

# 3. Install client tools
cd llm-d/guides/prereq/client-setup
./install-deps.sh
```

## Key Configuration Changes

| Setting | Intel Data Center GPU Max 1550 | Intel BMG |
|---------|-------------------------------|-----------|
| GPU Resource | `gpu.intel.com/i915` | `gpu.intel.com/xe` |
| UCX Transport | RDMA possible | `tcp` only |
| Values File | `values_xpu.yaml` | `values_bmg.yaml` |

## Quick Deploy: Inference Scheduling

```bash
# Setup
export NAMESPACE=llm-d-bmg
export HF_TOKEN=your-token-here

# Create namespace and secret
kubectl create namespace ${NAMESPACE}
kubectl create secret generic llm-d-hf-token \
  --from-literal="HF_TOKEN=${HF_TOKEN}" \
  --namespace ${NAMESPACE}

# Install Gateway
cd guides/prereq/gateway-provider
./install-gateway-provider-dependencies.sh
helmfile apply -f istio.helmfile.yaml

# Deploy with BMG values
cd ../inference-scheduling
helmfile apply -n ${NAMESPACE} \
  --state-values-set modelservice.valuesFiles='{ms-inference-scheduling/values_bmg.yaml}'

# Install HTTPRoute
kubectl apply -f httproute.yaml -n ${NAMESPACE}
```

## Quick Deploy: P/D Disaggregation

```bash
# Setup (same as above)
export NAMESPACE=llm-d-pd-bmg
export HF_TOKEN=your-token-here

kubectl create namespace ${NAMESPACE}
kubectl create secret generic llm-d-hf-token \
  --from-literal="HF_TOKEN=${HF_TOKEN}" \
  --namespace ${NAMESPACE}

# Deploy with BMG values
cd guides/pd-disaggregation
helmfile apply -n ${NAMESPACE} \
  --state-values-set modelservice.valuesFiles='{ms-pd/values_bmg.yaml}'

# Install HTTPRoute
kubectl apply -f httproute.yaml -n ${NAMESPACE}
```

## BMG-Specific Values Files

Ready-to-use values files are available:

- **Inference Scheduling**: `guides/inference-scheduling/ms-inference-scheduling/values_bmg.yaml`
- **P/D Disaggregation**: `guides/pd-disaggregation/ms-pd/values_bmg.yaml`

## Essential YAML Snippet

For custom deployments, use this template:

```yaml
accelerator:
  type: intel
  resources:
    intel: "gpu.intel.com/xe"  # Key change for BMG

containers:
- name: "vllm"
  image: ghcr.io/llm-d/llm-d-xpu:v0.3.1
  env:
    - name: UCX_TLS
      value: "tcp"  # Required for BMG
  resources:
    limits:
      gpu.intel.com/xe: 1  # Use 'xe' not 'i915'
    requests:
      gpu.intel.com/xe: 1
```

## NIXL Configuration (P/D Only)

```yaml
# Routing proxy
routing:
  proxy:
    connector: nixlv2
    secure: false

# Container environment
env:
  - name: UCX_TLS
    value: "tcp"
  - name: VLLM_NIXL_SIDE_CHANNEL_HOST
    valueFrom:
      fieldRef:
        fieldPath: status.podIP
  - name: VLLM_NIXL_SIDE_CHANNEL_PORT
    value: "5557"

# vLLM args
args:
  - "--kv-transfer-config"
  - '{"kv_connector":"NixlConnector", "kv_role":"kv_both", "kv_buffer_device":"cpu"}'
```

## Verification Commands

```bash
# Check deployment
helm list -n ${NAMESPACE}
kubectl get pods -n ${NAMESPACE}

# View logs
POD=$(kubectl get pods -n ${NAMESPACE} -l app.kubernetes.io/component=modelservice -o jsonpath='{.items[0].metadata.name}')
kubectl logs -n ${NAMESPACE} ${POD} -c vllm -f

# Test inference
kubectl port-forward -n ${NAMESPACE} service/infra-*-inference-gateway-istio 8086:80 &
curl http://localhost:8086/health

curl -X POST "http://localhost:8086/v1/chat/completions" \
  -H "Content-Type: application/json" \
  -d '{"model":"Qwen/Qwen3-0.6B","messages":[{"role":"user","content":"Hello"}],"max_tokens":50}'
```

## Common Issues

| Issue | Cause | Fix |
|-------|-------|-----|
| Pods pending | GPU not detected | Check GPU plugin: `kubectl get pods -n kube-system \| grep intel-gpu` |
| Wrong resource type | Using i915 instead of xe | Change all `gpu.intel.com/i915` → `gpu.intel.com/xe` |
| NIXL errors | Wrong transport | Set `UCX_TLS=tcp` in env |
| OOM errors | Insufficient memory | Increase `sizeLimit` in dshm volume |

## Resource Recommendations

**Inference Scheduling:**
- Start with 2 replicas
- 32Gi memory per replica
- 8 CPU cores per replica

**P/D Disaggregation:**
- 1 decode replica (16 CPU, 64Gi memory)
- 3 prefill replicas (8 CPU, 64Gi memory each)
- Total: 4 BMG GPUs

## Next Steps

See full guide: [docs/intel-bmg-setup.md](./intel-bmg-setup.md)

## Support

- **Intel XPU Maintainer**: Yuan Wu (yuan.wu@intel.com)
- **llm-d Slack**: https://llm-d.ai/slack
- **GitHub Issues**: https://github.com/llm-d/llm-d/issues
