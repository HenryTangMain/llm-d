# Intel BMG (Battlemage) Setup Guide for llm-d

This guide provides complete instructions for deploying llm-d on Intel BMG (Battlemage G21) GPUs.

## Table of Contents

- [Overview](#overview)
- [Hardware Requirements](#hardware-requirements)
- [Prerequisites](#prerequisites)
- [Key Differences from Intel Data Center GPU Max 1550](#key-differences-from-intel-data-center-gpu-max-1550)
- [Setup Steps](#setup-steps)
  - [1. Install Intel GPU Device Plugin](#1-install-intel-gpu-device-plugin)
  - [2. Build Container Image](#2-build-container-image-optional)
  - [3. Configure Helm Values](#3-configure-helm-values)
  - [4. Deploy llm-d](#4-deploy-llm-d)
- [NIXL Configuration for BMG](#nixl-configuration-for-bmg)
- [Deployment Examples](#deployment-examples)
- [Verification](#verification)
- [Troubleshooting](#troubleshooting)

## Overview

Intel BMG (Battlemage G21) GPUs are supported in llm-d with some configuration changes compared to Intel Data Center GPU Max 1550. The primary differences are:

- **GPU Resource Type**: `gpu.intel.com/xe` (instead of `gpu.intel.com/i915`)
- **Network Transport**: TCP-based communication (instead of RDMA)
- **Use Cases**: Suitable for inference-scheduling and pd-disaggregation workloads

## Hardware Requirements

- **GPU**: Intel Corporation Battlemage G21 or compatible Intel Xe architecture GPU
- **Memory**: At least 8GB system RAM per GPU
- **Storage**: Minimum 50GB available disk space
- **Network**: Standard TCP/IP networking (no special RDMA requirements)

## Prerequisites

### Software Requirements

- **Kubernetes**: v1.28.0 or newer
- **kubectl**: Cluster admin privileges
- **Helm**: v3.12.0 or newer
- **Helmfile**: v1.1.0 or newer
- **Docker/Podman**: For building container images (optional)

### Install Client Tools

```bash
cd llm-d
./guides/prereq/client-setup/install-deps.sh
```

### Create Kubernetes Cluster (if needed)

If you don't have a Kubernetes cluster:

```bash
# Install Kind
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.20.0/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind

# Create cluster
kind create cluster --name llm-d-bmg --image kindest/node:v1.28.15

# Verify
kubectl cluster-info
kubectl get nodes
```

## Key Differences from Intel Data Center GPU Max 1550

| Component | Intel Data Center GPU Max 1550 | Intel BMG (Battlemage G21) |
|-----------|-------------------------------|---------------------------|
| GPU Resource | `gpu.intel.com/i915` | `gpu.intel.com/xe` |
| UCX Transport | Can use RDMA | `tcp` only |
| Device Plugin | intel-device-plugins-for-kubernetes | Same |
| Container Image | Same Dockerfile.xpu | Same |
| NIXL Support | Yes | Yes |

## Setup Steps

### 1. Install Intel GPU Device Plugin

The Intel GPU Device Plugin enables Kubernetes to discover and schedule workloads on Intel GPUs:

```bash
kubectl apply -k 'https://github.com/intel/intel-device-plugins-for-kubernetes/deployments/gpu_plugin?ref=v0.32.1'
```

Verify the plugin is running:

```bash
kubectl get pods -n kube-system | grep intel-gpu
```

Check GPU resources are available:

```bash
kubectl get nodes -o json | jq '.items[].status.capacity | select(.["gpu.intel.com/xe"] != null)'
```

### 2. Build Container Image (Optional)

If using pre-built images, skip to step 3. Otherwise, build the Intel XPU image:

#### Option A: Using Make (Standard)

```bash
cd llm-d
make image-build DEVICE=xpu VERSION=v0.3.1
```

#### Option B: Using vLLM Source (Custom)

```bash
git clone https://github.com/vllm-project/vllm.git
cd vllm
git checkout v0.11.0
docker build -f docker/Dockerfile.xpu \
  -t ghcr.io/llm-d/llm-d-xpu-dev:v0.3.1 \
  --shm-size=4g .
```

#### Load Image into Kind (if using Kind)

```bash
kind load docker-image ghcr.io/llm-d/llm-d-xpu:v0.3.1 --name llm-d-bmg

# Verify
docker exec -it llm-d-bmg-control-plane crictl images | grep llm-d
```

### 3. Configure Helm Values

The critical step for BMG is updating GPU resource types from `i915` to `xe`.

#### For Inference Scheduling

Edit `guides/inference-scheduling/ms-inference-scheduling/values_xpu.yaml`:

```yaml
accelerator:
  type: intel
  resources:
    intel: "gpu.intel.com/xe"  # Changed from i915

containers:
- name: "vllm"
  image: ghcr.io/llm-d/llm-d-xpu:v0.3.1
  resources:
    limits:
      gpu.intel.com/xe: 1  # Changed from i915
    requests:
      gpu.intel.com/xe: 1  # Changed from i915
```

#### For P/D Disaggregation

Edit `guides/pd-disaggregation/ms-pd/values_xpu.yaml`:

```yaml
accelerator:
  type: intel
  resources:
    intel: "gpu.intel.com/xe"  # Changed from i915

decode:
  containers:
  - name: "vllm"
    resources:
      limits:
        gpu.intel.com/xe: 1  # Changed from i915
      requests:
        gpu.intel.com/xe: 1  # Changed from i915

prefill:
  containers:
  - name: "vllm"
    resources:
      limits:
        gpu.intel.com/xe: 1  # Changed from i915
      requests:
        gpu.intel.com/xe: 1  # Changed from i915
```

#### Create BMG-Specific Values File (Recommended)

For a cleaner approach, create a new file `values_bmg.yaml`:

```yaml
# Intel BMG configuration for Inference Scheduling
accelerator:
  type: intel
  resources:
    intel: "gpu.intel.com/xe"

modelArtifacts:
  uri: "hf://Qwen/Qwen3-0.6B"
  size: 10Gi
  authSecretName: "llm-d-hf-token"
  name: "Qwen/Qwen3-0.6B"

routing:
  servicePort: 8000

containers:
- name: "vllm"
  image: ghcr.io/llm-d/llm-d-xpu:v0.3.1
  modelCommand: vllmServe
  args:
    - "--max-model-len"
    - "8192"
    - "--disable-log-requests"
  env:
    - name: VLLM_WORKER_MULTIPROC_METHOD
      value: "spawn"
  resources:
    limits:
      memory: 32Gi
      cpu: "8"
      gpu.intel.com/xe: 1
    requests:
      memory: 32Gi
      cpu: "8"
      gpu.intel.com/xe: 1
  volumeMounts:
    - name: dshm
      mountPath: /dev/shm
  mountModelVolume: true

volumes:
  - name: dshm
    emptyDir:
      medium: Memory
      sizeLimit: 2Gi
```

### 4. Deploy llm-d

#### Setup Environment

```bash
export NAMESPACE=llm-d-bmg
export HF_TOKEN=your-huggingface-token

# Create namespace
kubectl create namespace ${NAMESPACE}

# Create HuggingFace token secret
kubectl create secret generic llm-d-hf-token \
  --from-literal="HF_TOKEN=${HF_TOKEN}" \
  --namespace ${NAMESPACE}
```

#### Install Gateway Provider

```bash
cd guides/prereq/gateway-provider

# Install Gateway API dependencies
./install-gateway-provider-dependencies.sh

# Deploy Istio Gateway control plane
helmfile apply -f istio.helmfile.yaml
```

#### Deploy Inference Scheduling (Recommended for BMG)

```bash
cd guides/inference-scheduling

# Deploy with XPU configuration
helmfile apply -e xpu -n ${NAMESPACE}

# Or with custom BMG values (if you created values_bmg.yaml)
helmfile apply -n ${NAMESPACE} \
  --state-values-set modelservice.valuesFiles='{ms-inference-scheduling/values_bmg.yaml}'
```

#### Deploy P/D Disaggregation (Advanced)

```bash
cd guides/pd-disaggregation

# Deploy with XPU configuration
helmfile apply -e xpu -n ${NAMESPACE}
```

#### Install HTTPRoute

For Istio or kgateway:

```bash
kubectl apply -f httproute.yaml -n ${NAMESPACE}
```

## NIXL Configuration for BMG

NIXL (Network Inference eXchange Library) is used for KV cache transfer in P/D disaggregation. It comes pre-installed in the XPU container image.

### Key NIXL Settings for BMG

**1. UCX Transport Layer**

BMG requires TCP transport (not RDMA):

```yaml
env:
  - name: UCX_TLS
    value: "tcp"
```

**2. NIXL Side Channel Configuration**

```yaml
env:
  - name: VLLM_NIXL_SIDE_CHANNEL_HOST
    valueFrom:
      fieldRef:
        fieldPath: status.podIP
  - name: VLLM_NIXL_SIDE_CHANNEL_PORT
    value: "5557"
```

**3. vLLM KV Transfer Configuration**

```yaml
args:
  - "--kv-transfer-config"
  - '{"kv_connector":"NixlConnector", "kv_role":"kv_both", "kv_buffer_device":"cpu"}'
```

**4. Routing Proxy Configuration**

```yaml
routing:
  proxy:
    image: ghcr.io/llm-d/llm-d-routing-sidecar:v0.3.0
    connector: nixlv2
    secure: false
```

### Why TCP for BMG?

- Intel BMG lacks dedicated high-speed interconnects (InfiniBand/RoCE)
- TCP is the reliable transport for standard networking
- RDMA requires specialized hardware (available on Data Center GPU Max with proper networking)

## Deployment Examples

### Example 1: Simple Inference Scheduling

Minimal configuration for testing:

```bash
export NAMESPACE=llm-d-test
kubectl create namespace ${NAMESPACE}

# Create empty HF token for public models
kubectl create secret generic llm-d-hf-token \
  --from-literal="HF_TOKEN=" \
  --namespace ${NAMESPACE}

cd guides/inference-scheduling
helmfile apply -e xpu -n ${NAMESPACE}
```

### Example 2: P/D Disaggregation with Custom Model

```bash
export NAMESPACE=llm-d-pd-bmg
export HF_TOKEN=hf_your_token_here

kubectl create namespace ${NAMESPACE}
kubectl create secret generic llm-d-hf-token \
  --from-literal="HF_TOKEN=${HF_TOKEN}" \
  --namespace ${NAMESPACE}

cd guides/pd-disaggregation
helmfile apply -e xpu -n ${NAMESPACE}
```

## Verification

### Check Helm Releases

```bash
helm list -n ${NAMESPACE}
```

Expected output:
```
NAME                NAMESPACE       STATUS      CHART
gaie-<suffix>       llm-d-bmg       deployed    inferencepool-v1.0.1
infra-<suffix>      llm-d-bmg       deployed    llm-d-infra-v1.3.0
ms-<suffix>         llm-d-bmg       deployed    llm-d-modelservice-v0.2.11
```

### Check Pod Status

```bash
kubectl get pods -n ${NAMESPACE}
```

All pods should be in `Running` state with `READY` status.

### View vLLM Logs

```bash
# Get pod name
POD_NAME=$(kubectl get pods -n ${NAMESPACE} -l app.kubernetes.io/component=modelservice -o jsonpath='{.items[0].metadata.name}')

# View logs
kubectl logs -n ${NAMESPACE} ${POD_NAME} -c vllm -f
```

Look for:
- Model loading completion
- GPU detection: "Using Intel XPU"
- Server startup: "Uvicorn running on http://0.0.0.0:8000"

### Test Inference

```bash
# Port forward to gateway service
kubectl port-forward -n ${NAMESPACE} \
  service/infra-*-inference-gateway-istio 8086:80 &

# Health check
curl -X GET "http://localhost:8086/health"

# Test inference
curl -X POST "http://localhost:8086/v1/chat/completions" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "Qwen/Qwen3-0.6B",
    "messages": [
      {
        "role": "user",
        "content": "Hello! Test inference on Intel BMG GPU."
      }
    ],
    "max_tokens": 50,
    "temperature": 0.7
  }'
```

## Troubleshooting

### Issue: Pods stuck in Pending state

**Cause**: GPU resources not available

**Solution**:
```bash
# Check if GPU plugin is running
kubectl get pods -n kube-system | grep intel-gpu

# Check node capacity
kubectl describe nodes | grep -A 5 "Capacity:"

# Verify xe resources are advertised
kubectl get nodes -o json | jq '.items[].status.capacity'
```

### Issue: vLLM fails to start with GPU error

**Cause**: Wrong GPU resource type

**Solution**: Verify you're using `gpu.intel.com/xe` not `gpu.intel.com/i915`

```bash
kubectl get pod <pod-name> -n ${NAMESPACE} -o yaml | grep "gpu.intel.com"
```

### Issue: NIXL connection failures in P/D disaggregation

**Cause**: Incorrect UCX transport configuration

**Solution**: Ensure `UCX_TLS=tcp` is set:

```bash
kubectl get pod <pod-name> -n ${NAMESPACE} -o yaml | grep UCX_TLS
```

### Issue: Model download failures

**Cause**: Missing or invalid HuggingFace token

**Solution**:
```bash
# Verify secret exists
kubectl get secret llm-d-hf-token -n ${NAMESPACE}

# Check secret content
kubectl get secret llm-d-hf-token -n ${NAMESPACE} -o jsonpath='{.data.HF_TOKEN}' | base64 -d
```

### Issue: Out of memory errors

**Cause**: Insufficient shared memory

**Solution**: Increase `sizeLimit` in dshm volume:

```yaml
volumes:
  - name: dshm
    emptyDir:
      medium: Memory
      sizeLimit: 4Gi  # Increase from 2Gi
```

### Debug Commands

```bash
# Check all resources in namespace
kubectl get all -n ${NAMESPACE}

# Describe pod for events
kubectl describe pod <pod-name> -n ${NAMESPACE}

# Check Gateway status
kubectl get gateway -n ${NAMESPACE} -o yaml

# Check HTTPRoute status
kubectl get httproute -n ${NAMESPACE} -o yaml

# Check InferencePool status
kubectl get inferencepool -n ${NAMESPACE} -o yaml
```

## Performance Tuning

### For Inference Scheduling

- Start with 1-2 replicas per BMG GPU
- Adjust `--max-model-len` based on available GPU memory
- Monitor metrics and scale replicas as needed

### For P/D Disaggregation

- Typical ratio: 3-4 prefill replicas : 1 decode replica
- Adjust based on input/output sequence length ratio
- Monitor KV cache transfer latency

### Resource Limits

```yaml
resources:
  limits:
    memory: 32Gi      # Adjust based on model size
    cpu: "8"          # Scale with concurrency needs
    gpu.intel.com/xe: 1
  requests:
    memory: 32Gi
    cpu: "8"
    gpu.intel.com/xe: 1
```

## Next Steps

- Review [llm-d guides](../guides/README.md) for advanced configurations
- Join [llm-d Slack](https://llm-d.ai/slack) for community support
- Check [accelerator documentation](./accelerators/README.md) for hardware-specific details
- Explore [monitoring setup](./monitoring/README.md) for production observability

## Additional Resources

- **Intel GPU Device Plugin**: https://github.com/intel/intel-device-plugins-for-kubernetes
- **vLLM XPU Support**: https://docs.vllm.ai/en/latest/getting_started/xpu-installation.html
- **llm-d Documentation**: https://www.llm-d.ai
- **Intel XPU Maintainer**: Yuan Wu (yuan.wu@intel.com)
