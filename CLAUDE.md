# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

llm-d is a Kubernetes-native distributed inference serving stack providing well-lit paths for serving large generative AI models at scale. It integrates vLLM as the default model server, Inference Gateway (IGW) for intelligent scheduling, and Kubernetes as the infrastructure orchestrator.

**Core Principles:**
- Keep it simple - users rapidly achieve running inference along a few well-lit paths
- Composition over configurability - components connect at API boundaries
- Fast iteration - experimental code and features are encouraged as long as they are opt-in
- Respect upstreams - vLLM and inference-gateway are where code changes start, no forks
- Ship to production - core code has high review, test, and reliability bar
- Hyrum's Law is real - no regression on published APIs or breaking changes

## Repository Structure

This is a **meta-repository** containing:
- **Documentation and guides** in `/guides` and `/docs`
- **Docker build configurations** in `/docker` for vLLM container images
- **Scripts and automation** in `/scripts` and `/hooks`
- **Infrastructure provider docs** in `/docs/infra-providers`
- **Patches** in `/patches` for NVSHMEM and other dependencies

**Component Repositories:**
- Core components: Located in `llm-d` GitHub org (production-ready, follows API deprecation process)
- Incubating components: Located in `llm-d-incubation` GitHub org (experimental, rapid iteration)

## Key Development Commands

### Container Image Building

**Build images** (uses docker/podman/buildah automatically):
```bash
# Build CUDA image (default)
make image-build

# Build for specific device
make image-build DEVICE=cuda        # NVIDIA GPUs
make image-build DEVICE=xpu         # Intel XPUs
make image-build DEVICE=cuda-efa    # AWS with EFA support

# Build with specific version
make image-build VERSION=v0.3.0

# Build dev vs prod images
make image-build BUILD_TYPE=dev     # Development build
make image-build BUILD_TYPE=prod    # Production build
```

**Push images**:
```bash
make image-push
make image-push DEVICE=xpu VERSION=v0.3.0
```

**Multi-arch build and push** (requires buildah or docker buildx):
```bash
make buildah-build
```

### Pre-commit and Linting

**Install git hooks**:
```bash
make install-hooks
```

**Pre-commit checks include:**
- `shellcheck` for shell scripts in `docker/scripts/`
- `hadolint` for Dockerfiles
- Custom linting for environment variables consistency
- `markdownlint` for markdown files
- `yamllint` for YAML files

**Run pre-commit manually**:
```bash
pre-commit run --all-files
```

### Testing

This repository primarily contains infrastructure and documentation. Testing is focused on:
- Docker image builds
- Linting and validation scripts
- E2E tests via GitHub Actions (see `.github/workflows/`)

**Component-specific tests** are located in their respective repositories (inference-scheduler, modelservice charts, etc).

## Architecture and Components

### Three Well-Lit Paths

1. **Intelligent Inference Scheduling** (`guides/inference-scheduling/`)
   - vLLM behind Inference Gateway with load-aware and prefix-cache aware balancing
   - Default configuration for most deployments
   - Supports precise prefix cache aware routing

2. **Prefill/Decode Disaggregation** (`guides/pd-disaggregation/`)
   - Splits inference into prefill servers (prompts) and decode servers (responses)
   - Reduces TTFT and improves TPOT predictability
   - Uses NIXL for KV cache transfer over fast interconnects (IB/RoCE RDMA, TPU ICI)

3. **Wide Expert-Parallelism** (`guides/wide-ep-lws/`)
   - For very large MoE models like DeepSeek-R1
   - Scales with Data Parallelism and Expert Parallelism
   - Requires fast accelerator networks

### Key Technologies

- **vLLM**: Default model server and inference engine
- **Inference Gateway (IGW)**: Request scheduler with Envoy-based load balancing
- **NIXL**: KV cache transfer library for disaggregation
- **UCX**: Communication framework for RDMA and high-speed networks
- **Kubernetes**: Infrastructure orchestrator (1.29+)

### Supported Hardware

- **NVIDIA GPUs**: A100, L4, H100, H200, or newer
- **AMD GPUs**: MI250 or newer
- **Google TPUs**: v5e or newer
- **Intel XPUs**: Data Center GPU Max (Ponte Vecchio) series

## Docker Image Build Architecture

Images are built using multi-stage Dockerfiles in `/docker`:
- `Dockerfile.cuda` - NVIDIA GPUs (default)
- `Dockerfile.xpu` - Intel XPUs
- `Dockerfile.aws` - AWS with EFA support
- `Dockerfile.gke` - GKE with TPU support

**Build process:**
1. Builder stage: Installs build dependencies, compiles custom kernels (pplx, deepep, deepgemm), builds UCX and NIXL from source
2. Runtime stage: Installs vLLM (from precompiled wheels by default), runtime dependencies, and custom components

**Key environment variables** are documented in `/scripts/ENVVARS.md` and validated via custom linting.

**Custom kernels supported:**
- `pplx` - All2All backend (set `VLLM_ALL2ALL_BACKEND=pplx`)
- `deepep` - DeepSeek kernels for high throughput (`deepep_high_throughput`) and low latency (`deepep_low_latency`)
- `deepgemm` - Custom GEMM kernels

## Deployment and Operations

### Deploying guides using Helmfile

Most guides use helmfile for composition:
```bash
cd guides/inference-scheduling
export NAMESPACE=llm-d
helmfile apply -n ${NAMESPACE}

# With specific gateway provider
helmfile apply -e kgateway -n ${NAMESPACE}
helmfile apply -e istio -n ${NAMESPACE}
helmfile apply -e gke -n ${NAMESPACE}

# For specific hardware
helmfile apply -e xpu -n ${NAMESPACE}
helmfile apply -e gke_tpu -n ${NAMESPACE}
```

**Important:** Use short namespace names to avoid hostname length issues (Linux max 64 chars).

### Prerequisites for deployment

Before deploying any guide:
1. Install required client tools (kubectl, helm, helmfile, etc) - see `guides/prereq/client-setup/`
2. Configure cluster infrastructure - see `guides/prereq/infrastructure/`
3. Deploy Gateway control plane - see `guides/prereq/gateway-provider/`
4. Create HuggingFace token secret: `kubectl create secret generic llm-d-hf-token --from-literal=HF_TOKEN=<token>`
5. Install monitoring stack - see `docs/monitoring/`

## Contributing and Development Process

### Contribution Types

1. **Features with public APIs or new components** - Require project proposal in `docs/proposals/` using the template
2. **Fixes, issues, and bugs** - Clear description, reproduction steps, and solution explanation

### Code Review Requirements

- All code changes via pull requests (no direct pushes)
- All changes reviewed and approved by maintainer other than author
- All repos gate merges on compilation and passing tests
- All experimental features off by default with explicit opt-in
- DCO sign-off required: `git commit -s`

### Experimental Features

**Naming convention:** Must include `experimental` in name (e.g., `--experimental-disaggregation-v2=true`)

**Requirements:**
- Clear identification in code and docs
- Default to off with explicit enablement
- Best effort support only
- No stigma to experimental status

### API Changes and Deprecation

- **No breaking changes** once in GA release (non-experimental)
- Includes all protocols, API endpoints, internal APIs, command-line flags
- Exception: Bug fixes not impacting significant consumers
- All protocols/APIs must be versionable with clear compatibility requirements

### Testing Layers

When contributing changes, identify the testing layer:

1. **Container Image Changes** - Test in multiple guides:
   - `inference-scheduling` guide
   - `precise-kv-cache-aware` example
   - `pd-disaggregation` (covers DeepSeek kernels)
   - `wide-ep-lws` (covers DeepSeek kernels)
   - Run guidellm benchmark for performance regression

2. **Deployment Changes** - Verify Kubernetes resources:
   - InferencePool exists and configured
   - Gateway has address and proper status
   - HTTPRoute attached to Gateway properly
   - vLLM pods running
   - PodMonitors deployed (if metrics enabled)

3. **Backend/Kernel Changes** - Set proper environment variables:
   - For pplx: `VLLM_ALL2ALL_BACKEND=pplx`
   - For DeepSeek: prefill uses `deepep_high_throughput`, decode uses `deepep_low_latency`
   - For UCX/NIXL: test in `pd-disaggregation` or `wide-ep-lws`

## Environment Variables and Configuration

**Critical environment variables** are documented in `/scripts/ENVVARS.md` and enforced via linting.

**Common variables:**
- `DEVICE` - Target device (cuda, xpu, cuda-efa)
- `BUILD_TYPE` - dev or prod
- `VERSION` - Image version tag
- `NAMESPACE` - Kubernetes namespace for deployments
- `RELEASE_NAME_POSTFIX` - Postfix for helm release names (for concurrent installs)

## Release Process

- Releases tracked via GitHub Releases
- Images published to `ghcr.io/llm-d`
- Guides kept current as living docs
- Helm charts versioned independently in component repos

## Community and Communication

- **Slack**: [llm-d.slack.com](https://llm-d.slack.com) - primary discussion
- **Weekly standup**: Wednesdays at 12:30 PM ET - see [public calendar](https://red.ht/llm-d-public-calendar)
- **Google Group**: llm-d-contributors@googlegroups.com - document sharing
- **Issues**: Report project-scoped bugs in [llm-d/llm-d](https://github.com/llm-d/llm-d)
- **SIGs**: Join Special Interest Groups for domain-specific collaboration - see SIGS.md

## Important Files to Review

- `PROJECT.md` - Project governance and process
- `CONTRIBUTING.md` - Detailed contribution guidelines
- `ONBOARDING.md` - On-boarding/off-boarding policy
- `PR_SIGNOFF.md` - DCO sign-off configuration
- `SIGS.md` - Special Interest Groups
- `guides/README.md` - Well-lit paths overview
- `docs/proposals/` - Project proposals and design docs
