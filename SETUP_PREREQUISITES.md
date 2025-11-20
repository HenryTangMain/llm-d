# Prerequisites Setup for Inference-Scheduling (Intel XPU)

Follow these steps to set up prerequisites for deploying llm-d inference-scheduling.

## Step 1: Install Gateway API CRDs

```bash
cd /scratch2/ytang/llm-d/guides/prereq/gateway-provider
./install-gateway-provider-dependencies.sh
```

This installs:
- Gateway API CRDs (v1.3.0)
- Gateway API Inference Extension CRDs (v1.0.1)

## Step 2: Deploy Istio Gateway Control Plane

```bash
export PATH="$HOME/bin:$PATH"
cd /scratch2/ytang/llm-d/guides/prereq/gateway-provider
helmfile apply -f istio.helmfile.yaml
```

This deploys Istio as the Gateway provider.

## Step 3: Create HuggingFace Token Secret

```bash
export NAMESPACE=llm-d
export HF_TOKEN="your-huggingface-token-or-empty-for-public-models"

# Create namespace
kubectl create namespace ${NAMESPACE}

# Create secret
kubectl create secret generic llm-d-hf-token \
  --from-literal="HF_TOKEN=${HF_TOKEN}" \
  --namespace ${NAMESPACE}
```

**Note:** Use empty string for public models, or your actual HF token for gated models.

## Step 4: Deploy Inference-Scheduling

```bash
cd /scratch2/ytang/llm-d/guides/inference-scheduling
export PATH="$HOME/bin:$PATH"
export NAMESPACE=llm-d

helmfile apply -e xpu -n ${NAMESPACE}
```

## Step 5: Verify Deployment

```bash
# Check helm releases
helm list -n ${NAMESPACE}

# Check pods
kubectl get pods -n ${NAMESPACE}

# Monitor pod logs
kubectl logs -n ${NAMESPACE} -l app=vllm -f
```

## Step 6: Test Inference

```bash
# Port forward to gateway
kubectl port-forward -n ${NAMESPACE} service/infra-inference-scheduling-inference-gateway-istio 8086:80 &

# Test health
curl http://localhost:8086/health

# Test completion
curl -X POST "http://localhost:8086/v1/chat/completions" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "facebook/opt-125m",
    "messages": [{"role": "user", "content": "Hello!"}],
    "max_tokens": 50
  }'
```

## Troubleshooting

**If helmfile hangs:**
- Check network/proxy: `curl -I https://llm-d-incubation.github.io/llm-d-modelservice/`
- Run with debug: `helmfile apply -e xpu -n ${NAMESPACE} --debug`

**If pods don't start:**
- Check GPU resources: `kubectl describe node | grep gpu.intel.com`
- Check pod events: `kubectl describe pod -n ${NAMESPACE} <pod-name>`
- Check logs: `kubectl logs -n ${NAMESPACE} <pod-name>`

**For Intel BMG GPU (Arc B580):**
You may need to update resource requests from `gpu.intel.com/i915` to `gpu.intel.com/xe` in the values file.

## Notes

- This setup uses 1 Intel Arc B580 GPU
- Default model is small for testing
- Adjust model and resources in helmfile values as needed
