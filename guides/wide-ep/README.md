# Well-lit Path: Wide Expert Parallelism (EP/DP)

## Overview

This guide demonstrates how to deploy DeepSeek-R1 using vLLM's P/D disaggregation support with NIXL in a wide expert parallel pattern. Wide Expert Parallelism (EP) enables efficient inference for large Mixture-of-Experts (MoE) models by distributing experts across multiple nodes while maintaining high throughput.

This guide supports two deployment variants depending on your orchestration requirements:

- **[LeaderWorkerSet (LWS)](README-lws.md)** - Kubernetes-native controller for multi-pod workloads with leader/worker coordination
- **[Grove](README-grove.md)** - Purpose-built AI workload orchestrator with gang scheduling, topology-aware placement, and integrated DRA support

## Choosing a Variant

Both LWS and Grove can orchestrate multi-node inference workloads, each with its own unique strengths and features tailored for different operational needs.

### LeaderWorkerSet (LWS)

Use the [LWS variant](README-lws.md) to have:

- **Simple leader/worker model** for straightforward multi-pod coordination
- **Validated with RDMA networking** including InfiniBand and RoCE

The example in this guide deploys DeepSeek-R1-0528 on validated H200/B200 configurations:
- 1 DP=16 Prefill Worker
- 1 DP=16 Decode Worker

### Grove

Use the [Grove variant](README-grove.md) to take advantage of advanced orchestration for multi-node inference across any hardware architecture (H100, H200, A100, B200, GB200 etc.):

- **Automatic topology-aware placement** to maximize GPU interconnect utilization across nodes
- **Hierarchical gang scheduling** to ensure prefill and decode units are scheduled as a functional set, preventing resource deadlocks
- **Multi-level, graceful scaling** where decode workers can scale independently of prefill units while maintaining system ratios
- **System-level lifecycle management** treating multi-component deployments as a single operational unit for recovery and updates while preserving the topology requirements

**Additional benefits on GB200/MNNVL**: On clusters with Multi-Node NVLink connectivity, Grove provides automatic ComputeDomain configuration, DRA-based resource management, and ensures pods are co-located within the same NVLink fabric for near-single-node performance across multiple nodes.

The example in this guide deploys DeepSeek-R1-NVFP4 on a validated GB200 configuration:
- 1 node for Prefill (4 GPUs)
- 8 nodes for Decode (1 leader + 7 workers, 32 GPUs total)

## Next Steps

Choose your deployment variant and follow the corresponding guide:

- [Wide EP with LeaderWorkerSet (LWS)](README-lws.md)
- [Wide EP with Grove](README-grove.md)

