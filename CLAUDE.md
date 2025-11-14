# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

<<<<<<< HEAD
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
=======
llm-d is a Kubernetes-native distributed inference serving stack for large generative AI models. The project provides tested "well-lit paths" (documented, tested, and benchmarked solutions) for deploying large language models at scale using vLLM, Inference Gateway (IGW), and Kubernetes.

**Key Components:**
- **vLLM**: Default model server and inference engine
- **Inference Gateway (IGW)**: Request scheduler and load balancer with custom intelligent routing
- **Kubernetes**: Infrastructure orchestrator and workload control plane

**Hardware Support:** NVIDIA GPUs (A100/L4+), AMD GPUs (MI250+), Google TPUs (v5e+), Intel XPUs (Ponte Vecchio+)

## Development Commands

### Container Image Building

Build container images for different hardware targets:

```bash
# Build CUDA image (default)
make image-build DEVICE=cuda VERSION=v0.2.1

# Build Intel XPU image
make image-build DEVICE=xpu VERSION=v0.2.1

# Build AWS EFA-enabled image
make image-build DEVICE=cuda-efa VERSION=v0.2.1

# Push images
make image-push DEVICE=cuda VERSION=v0.2.1

# Show environment variables
make env DEVICE=cuda
```

**Note:** Set `BUILD_TYPE=dev` for development builds (default) or `BUILD_TYPE=prod` for production builds.

### Multi-arch builds with buildah

For production releases supporting multiple architectures:

```bash
make buildah-build DEVICE=cuda VERSION=v0.2.1
```

### Pre-commit Hooks

The project uses pre-commit hooks for code quality. Install them with:

>>>>>>> 90c1ade (update.)
```bash
make install-hooks
```

<<<<<<< HEAD
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
=======
This sets `core.hooksPath` to the `hooks/` directory. Pre-commit checks include:
- Shell script linting (shellcheck)
- Dockerfile linting (hadolint)
- Environment variable validation
- YAML/JSON validation
- Markdown linting
- Trailing whitespace and end-of-file fixes

### Deployment Testing

After deploying changes:

```bash
make post-deploy-test
```

## Repository Structure

```
llm-d/
├── docker/               # Container image definitions
│   ├── Dockerfile.cuda   # NVIDIA GPU image
│   ├── Dockerfile.xpu    # Intel XPU image
│   ├── Dockerfile.aws    # AWS EFA-enabled image
│   ├── Dockerfile.gke    # GKE-specific image
│   ├── scripts/          # Container build scripts
│   └── packages/         # Custom packages built into images
├── guides/               # Well-lit path guides and examples
│   ├── inference-scheduling/        # IGW + vLLM with smart load balancing
│   ├── pd-disaggregation/           # Prefill/Decode disaggregation
│   ├── wide-ep-lws/                 # Wide Expert Parallelism for MoE models
│   ├── precise-prefix-cache-aware/  # Precise KV cache aware routing
│   ├── predicted-latency-based-scheduling/  # Experimental latency-based routing
│   ├── prefix-cache-storage/        # KV cache offloading configurations
│   ├── simulated-accelerators/      # vLLM simulator for testing
│   ├── recipes/                     # Standardized deployment recipes
│   └── prereq/                      # Prerequisites and setup guides
├── docs/                 # Project documentation
├── scripts/              # Utility scripts for linting and validation
├── hooks/                # Git hooks (use `make install-hooks`)
└── patches/              # Patches applied to upstream dependencies
```

## Well-Lit Paths

The repository provides three primary deployment patterns in `guides/`:

1. **Intelligent Inference Scheduling** (`inference-scheduling/`): Deploy vLLM with IGW for optimized load balancing, prefix-cache aware routing, and reduced tail latency.

2. **Prefill/Decode Disaggregation** (`pd-disaggregation/`): Split inference into prefill and decode phases on independent instances to reduce TTFT and improve TPOT predictability for large models (Llama-70B+) with long prompts.

3. **Wide Expert-Parallelism** (`wide-ep-lws/`): Deploy very large MoE models (DeepSeek-R1) with Data Parallelism and Expert Parallelism over fast networks.

Each guide contains:
- `README.md`: Detailed deployment instructions and configuration options
- `helmfile.yaml`: Helmfile composition for deploying the stack
- `values.yaml`: Helm chart values for model server and scheduler configuration
- Hardware-specific variants (e.g., `xpu`, `tpu`, `cuda`)

## Deployment Workflow

Guides use Helmfile to compose and deploy stacks:

```bash
# Set namespace (use short names to avoid hostname length issues)
export NAMESPACE=llm-d

# Deploy with default gateway (Istio)
cd guides/inference-scheduling
helmfile apply -n ${NAMESPACE}

# Deploy with specific gateway provider
helmfile apply -e kgateway -n ${NAMESPACE}

# Deploy for specific hardware
helmfile apply -e xpu -n ${NAMESPACE}        # Intel XPU
helmfile apply -e gke_tpu -n ${NAMESPACE}    # Google TPU

# Support concurrent installs with different release names
RELEASE_NAME_POSTFIX=test-2 helmfile apply -n ${NAMESPACE}
```

**Important:** When using long namespace names, pod hostnames may exceed Linux limits (64 chars). Use shorter namespace names and set `RELEASE_NAME_POSTFIX` if needed.

## Container Image Architecture

The Dockerfiles build vLLM-based inference containers with:
- vLLM with precompiled binaries from upstream wheels (default)
- UCX built from source (for RDMA/high-speed networking)
- NIXL built against UCX (for KV cache transfer)
- Custom kernels: `pplx`, `deepep`, `deepgemm` (selectable via `VLLM_ALL2ALL_BACKEND`)
- LMCache support (for KV cache offloading, upcoming)
- EFA support on AWS (Dockerfile.aws)

### Backend Selection

Set `VLLM_ALL2ALL_BACKEND` environment variable:
- `pplx`: Both prefill and decode (default for most workloads)
- `deepep_high_throughput`: Prefill with DeepSeek kernels
- `deepep_low_latency`: Decode with DeepSeek kernels

## Contributing and Code Review

All contributions require:
- **Project Proposal** for features with public APIs or new components (`docs/proposals/`)
- **DCO Sign-off** on all commits (`git commit -s`)
- **Code Review** by a maintainer (no direct pushes)
- **Passing Tests** before merge
- **Experimental Features** must be off by default with `experimental` in flag names

### Feature Testing Checklist

When testing container image changes:
- [ ] `inference-scheduler` guide
- [ ] `precise-kv-cache-aware` example
- [ ] `pd-disaggregation` example (covers DeepSeek kernels)
- [ ] `wide-ep-lws` example (covers DeepSeek kernels)
- [ ] Run guidellm benchmark for performance regression testing
- [ ] Test `pplx` backend
- [ ] Test DeepSeek kernels (prefill: `deepep_high_throughput`, decode: `deepep_low_latency`)

### Deployment Layer Testing

**Gateway/Infrastructure Changes:**
- Verify `InferencePool` exists: `kubectl get InferencePool.inference.networking.x-k8s.io`
- Check `Gateway` status: `kubectl get gateway -o yaml` (look for `address` and "Resource programmed" message)
- Verify `parametersRef` resource exists
- Check `HTTPRoute` status: `kubectl get httpRoute -o yaml | yq '.status.parents[]'` (look for "Route was valid")
- For Istio: verify `DestinationRule` exists

**Model Server Changes:**
- Ensure vLLM pods are running
- Verify `PodMonitor` resources deployed if metrics enabled

## Environment Variable Management

Environment variables in shell scripts and Dockerfiles are validated by custom linters:
- `scripts/lint-envvars.py`: Validates shell script environment variables
- `scripts/lint-dockerfile-envvars.py`: Validates Dockerfile environment variables against shell scripts

See `scripts/ENVVARS.md` for documentation on tracked environment variables.

## Experimental Features and Incubation

- Experimental features must be **off by default** and clearly labeled
- Flag naming convention: include `experimental` in name (e.g., `--experimental-disaggregation-v2`)
- Incubating components live in `llm-d-incubation` GitHub org before graduation to `llm-d`
- No breaking changes to GA APIs/protocols without versioning

## API Stability and Versioning

- Once in GA release, APIs/protocols cannot be removed or behavior changed
- All APIs must be versionable with clear forward/backward compatibility
- Breaking changes require new API version
- Applies to: protocols, API endpoints, internal APIs, CLI flags

## Community Resources

- **Slack**: [llm-d.ai/slack](https://llm-d.ai/slack) - Developer discussions
- **Weekly Meeting**: Wednesdays 12:30 PM ET ([calendar](https://red.ht/llm-d-public-calendar))
- **Google Group**: [llm-d-contributors@googlegroups.com](mailto:llm-d-contributors@googlegroups.com)
- **GitHub Issues**: Report project-scoped bugs in [llm-d/llm-d](https://github.com/llm-d/llm-d)
- **SIGs**: See [SIGS.md](SIGS.md) for Special Interest Groups
>>>>>>> 90c1ade (update.)
