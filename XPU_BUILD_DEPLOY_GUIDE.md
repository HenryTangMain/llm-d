# vLLM XPU Docker: Build, Deploy, and Use Guide

Complete guide for building, deploying, and testing the vLLM XPU Docker image on Intel Data Center GPU Max series.

## Prerequisites

- Intel Data Center GPU Max (Ponte Vecchio) hardware
- Docker installed with XPU device access
- Corporate proxy configured (if behind firewall)

## Step 1: Build the Docker Image

### Quick Build (Using Build Script)

```bash
cd /scratch2/ytang/llm-d

# Use the build script (bypasses buildx hanging issue)
/tmp/build-xpu.sh
```

### Manual Build (Alternative)

```bash
cd /scratch2/ytang/llm-d

# Build with legacy builder (DOCKER_BUILDKIT=0)
DOCKER_BUILDKIT=0 docker build \
  -t ghcr.io/llm-d/llm-d-xpu-dev:v0.2.1 \
  -f docker/Dockerfile.xpu .
```

### Build Notes

The Dockerfile has been modified to work with the legacy Docker builder:
- **ENV statement** moved after FROM (required by Docker spec)
- **Cache mounts** removed (`--mount=type=cache`) for legacy builder compatibility
- **Proxy settings** configured for Intel corporate network

Build time: ~30-60 minutes depending on network speed.

## Step 2: Deploy the Docker Container

### Run with Small Test Model

```bash
docker run -d --rm --privileged \
  --device=/dev/dri \
  $(for dev in /dev/mei*; do echo --device $dev; done) \
  --group-add video \
  --cap-add=SYS_ADMIN \
  --mount type=bind,source=/dev/dri/by-path,target=/dev/dri/by-path \
  --mount type=bind,source=/sys,target=/sys \
  --mount type=bind,source=/dev/bus,target=/dev/bus \
  --mount type=bind,source=/dev/char,target=/dev/char \
  -p 8000:8000 \
  --name vllm-xpu \
  ghcr.io/llm-d/llm-d-xpu-dev:v0.2.1 \
  --model facebook/opt-125m \
  --device xpu \
  --dtype float16
```

### Deployment Options

**Model Selection:**
- `facebook/opt-125m` - Small test model (~250MB, fast download)
- `microsoft/phi-2` - Better quality (~2.7GB)
- `Qwen/Qwen2.5-0.5B-Instruct` - Tiny instruction model
- `meta-llama/Llama-2-7b-hf` - Production quality (requires HuggingFace token)

**Using Local Model:**
```bash
docker run -d --rm --privileged \
  --device=/dev/dri \
  $(for dev in /dev/mei*; do echo --device $dev; done) \
  --group-add video \
  --cap-add=SYS_ADMIN \
  --mount type=bind,source=/dev/dri/by-path,target=/dev/dri/by-path \
  --mount type=bind,source=/sys,target=/sys \
  --mount type=bind,source=/dev/bus,target=/dev/bus \
  --mount type=bind,source=/dev/char,target=/dev/char \
  --mount type=bind,source=/path/to/local/model,target=/models \
  -p 8000:8000 \
  --name vllm-xpu \
  ghcr.io/llm-d/llm-d-xpu-dev:v0.2.1 \
  --model /models \
  --device xpu \
  --dtype float16
```

### Monitor Container Startup

```bash
# Follow logs
docker logs -f vllm-xpu

# Wait for this message:
# "INFO:     Application startup complete."
```

## Step 3: Generate Text with Curl

### Health Check

```bash
curl http://localhost:8000/ping
```

Expected output: `{"ping":"pong!"}`

### List Available Models

```bash
curl http://localhost:8000/v1/models
```

### Text Completion

```bash
curl http://localhost:8000/v1/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "facebook/opt-125m",
    "prompt": "Once upon a time",
    "max_tokens": 50,
    "temperature": 0.7
  }'
```

**Sample Response:**
```json
{
  "id": "cmpl-xxx",
  "object": "text_completion",
  "created": 1234567890,
  "model": "facebook/opt-125m",
  "choices": [
    {
      "index": 0,
      "text": " in a land far away...",
      "logprobs": null,
      "finish_reason": "length"
    }
  ]
}
```

### Chat Completion

```bash
curl http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "facebook/opt-125m",
    "messages": [
      {"role": "system", "content": "You are a helpful assistant."},
      {"role": "user", "content": "Tell me a short story about AI"}
    ],
    "max_tokens": 100,
    "temperature": 0.7
  }'
```

### Advanced Parameters

```bash
curl http://localhost:8000/v1/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "facebook/opt-125m",
    "prompt": "Explain quantum computing in simple terms:",
    "max_tokens": 150,
    "temperature": 0.8,
    "top_p": 0.95,
    "n": 1,
    "stream": false,
    "stop": ["\n\n"]
  }'
```

**Parameters:**
- `max_tokens`: Maximum tokens to generate
- `temperature`: Randomness (0.0-2.0, higher = more random)
- `top_p`: Nucleus sampling threshold
- `n`: Number of completions to generate
- `stream`: Enable streaming responses
- `stop`: Stop sequences

## Step 4: Container Management

### View Logs

```bash
docker logs vllm-xpu
docker logs -f vllm-xpu  # Follow mode
docker logs --tail 100 vllm-xpu  # Last 100 lines
```

### Stop Container

```bash
docker stop vllm-xpu
```

### Restart Container

```bash
docker restart vllm-xpu
```

### Remove Container

```bash
docker rm -f vllm-xpu
```

### Interactive Shell

```bash
docker exec -it vllm-xpu bash
```

## Troubleshooting

### Container Won't Start

Check XPU device access:
```bash
ls -la /dev/dri
```

Verify Intel GPU drivers:
```bash
clinfo
xpu-smi discovery
```

### Model Download Fails

Check proxy settings:
```bash
docker run -it --rm ghcr.io/llm-d/llm-d-xpu-dev:v0.2.1 bash
# Inside container:
env | grep -i proxy
curl -I https://huggingface.co
```

### Out of Memory

Reduce model size or adjust parameters:
```bash
# Use smaller model
--model facebook/opt-125m

# Or reduce max model length
--max-model-len 2048
```

### Performance Issues

Check XPU utilization:
```bash
xpu-smi dump -m 1
```

## Next Steps

### Push to Registry

```bash
# Using make
make image-push DEVICE=xpu VERSION=v0.2.1

# Or manually
docker push ghcr.io/llm-d/llm-d-xpu-dev:v0.2.1
```

### Deploy with Kubernetes

Use one of the llm-d guides:
```bash
cd guides/inference-scheduling
export NAMESPACE=llm-d
helmfile apply -e xpu -n ${NAMESPACE}
```

### Benchmark Performance

```bash
# Install guidellm
pip install guidellm

# Run benchmark
guidellm --target http://localhost:8000/v1 \
  --model facebook/opt-125m \
  --prompt-tokens 512 \
  --max-tokens 128
```

## References

- [vLLM Documentation](https://docs.vllm.ai/)
- [Intel Extension for PyTorch](https://intel.github.io/intel-extension-for-pytorch/)
- [llm-d Project](https://github.com/llm-d/llm-d)
- [OpenAI API Reference](https://platform.openai.com/docs/api-reference)

## Support

- **Issues**: [llm-d/llm-d Issues](https://github.com/llm-d/llm-d/issues)
- **Slack**: [llm-d.slack.com](https://llm-d.slack.com)
- **Weekly Standup**: Wednesdays at 12:30 PM ET

---

*Generated with Claude Code*
