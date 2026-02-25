# Well-lit Path: Wide Expert Parallelism (EP/DP) with Grove

## Overview

This guide demonstrates how to deploy DeepSeek-R1-NVFP4 using vLLM's P/D disaggregation support with NIXL in a wide expert parallel pattern with Grove. This guide utilizes NVIDIA MNNVL (Multi-Node NVLink) technology for fast GPU-to-GPU communication. This guide has been validated on:

* A GB200 rack utilizing 9 nodes, each with 4 GPUs

In this example, we will demonstrate a deployment of `nvidia/DeepSeek-R1-NVFP4` with:

- 1 node for Prefill (4 GPUs)
- 8 nodes for Decode (1 leader node + 7 worker nodes, 32 GPUs total)

## How Grove Orchestrates This Deployment

This deployment utilizes Grove's PodCliqueSet to coordinate 9 nodes as a single logical inference system. The following describes how Grove's primitives map to the requirements of a Wide-EP workload:

- **Hierarchical gang scheduling**: Grove ensures the minimum viable combination of prefill and decode units are scheduled together, preventing resource deadlocks and ensuring service readiness
- **Topology-aware placement**: Grove leverages cluster topology to place pods optimally within NVLink domains, minimizing inter-node latency for KV-cache transfers
- **Multilevel, graceful scaling**: PodClique and PodCliqueScalingGroup primitives enable independent scaling of prefill and decode workers while maintaining correct component ratios and system integrity
- **System-level lifecycle & recovery**: Grove treats multi-component systems as single operational units. Recovery and updates operate on complete service instances, ensuring workers properly reconnect to leaders after a restart and rolling updates preserve network topology
- **Role-aware orchestration**: Defines explicit startup ordering (e.g., ensuring decode leaders are ready before workers) and role-specific configurations within a single declarative PodCliqueSet
- **Native scheduler integration**: Grove automatically translates workload intent into PodGang resources for the [KAI Scheduler](https://github.com/NVIDIA/KAI-Scheduler). This enables the scheduler to execute topology-aware placement, hierarchical queuing, optimizing the entire scheduling cycle by ensuring resources are allocated according to the specific performance and availability requirements of the workload
- **Automatic MNNVL configuration**: Grove abstracts away ComputeDomain and ResourceClaimTemplate complexity - just deploy your PodCliqueSet and Grove handles the rest

Grove is designed from the ground up for the unique requirements of AI inference - particularly on next-generation hardware like GB200 where maximizing NVLink fabric utilization is essential for performance.

## Hardware Requirements

This guide requires an NVIDIA GB200 rack with 9 nodes (4 GPUs each, 36 GPUs total) and NVIDIA MNNVL connectivity for multi-node NVLink communication. Check `manifests/modelserver/grove/podCliqueSet.yaml` for detailed resource requirements.

## Prerequisites

- Have the [proper client tools installed on your local system](../prereq/client-setup/README.md) to use this guide.
- Ensure your cluster infrastructure is sufficient to [deploy high scale inference](../prereq/infrastructure/README.md)
  - You must have high speed inter-accelerator networking (NVIDIA MNNVL)
  - The pods leveraging inter-node EP must be deployed within the same networking domain
  - You have deployed the [Grove operator](../prereq/infrastructure/README.md#optional-install-grove-for-multi-host-inference) (required for this guide)
  - Optionally, you have deployed the [KAI Scheduler](../prereq/infrastructure/README.md#optional-install-kai-scheduler-for-ai-workload-scheduling) for enhanced AI workload scheduling
- Configure and deploy your [Gateway control plane](../prereq/gateway-provider/README.md)
- Have the [Monitoring stack](../../docs/monitoring/README.md) installed on your system
- Create a namespace for installation
  
  ```
  export NAMESPACE=llm-d-wide-ep # or any other namespace (shorter names recommended)
  kubectl create namespace ${NAMESPACE}
  ```

- [Create the `llm-d-hf-token` secret in your target namespace with the key `HF_TOKEN` matching a valid HuggingFace token](../prereq/client-setup/README.md#huggingface-token) to pull models
- [Choose an llm-d version](../prereq/client-setup/README.md#llm-d-version)

## Installation

```bash
cd guides/wide-ep/
```

### Deploy Model Servers

Deploy the Grove PodCliqueSet and ComputeDomain for NVIDIA MNNVL support:

```bash
kubectl apply -k ./manifests/modelserver/grove/ -n ${NAMESPACE}
```

### Deploy InferencePool

Select the provider-specific Helm command using the tabs below.

<!-- TABS:START -->

<!-- TAB:GKE:default -->
#### GKE
```bash
helm install llm-d-infpool \
  -n ${NAMESPACE} \
  -f ./manifests/inferencepool.values.yaml \
  --set "provider.name=gke" \
  --set "inferencePool.apiVersion=inference.networking.k8s.io/v1" \
  --set "inferenceExtension.monitoring.gke.enable=true" \
  oci://us-central1-docker.pkg.dev/k8s-staging-images/gateway-api-inference-extension/charts/inferencepool \
  --version v1.2.0-rc.1
```

<!-- TAB:Istio -->
#### Istio
```bash
helm install llm-d-infpool \
  -n ${NAMESPACE} \
  -f ./manifests/inferencepool.values.yaml \
  --set "provider.name=istio" \
  --set "inferenceExtension.monitoring.prometheus.enable=true" \
  oci://us-central1-docker.pkg.dev/k8s-staging-images/gateway-api-inference-extension/charts/inferencepool \
  --version v1.2.0-rc.1
```

<!-- TAB:Kgateway -->
#### Kgateway
```bash
helm install llm-d-infpool \
  -n ${NAMESPACE} \
  -f ./manifests/inferencepool.values.yaml \
  oci://us-central1-docker.pkg.dev/k8s-staging-images/gateway-api-inference-extension/charts/inferencepool \
  --version v1.2.0-rc.1
```

<!-- TABS:END -->

### Deploy Gateway and HTTPRoute

Deploy the Gateway and HTTPRoute using the [gateway recipe](../recipes/gateway/README.md).

### Gateway options

To see what gateway options are supported refer to our [gateway provider prereq doc](../prereq/gateway-provider/README.md#supported-providers). Gateway configurations per provider are tracked in the [gateway-configurations directory](../prereq/gateway-provider/common-configurations/).

You can also customize your gateway, for more information on how to do that see our [gateway customization docs](../../docs/customizing-your-gateway.md).

## Tuning Selective PD

As with PD, the `wide-ep` guide supports selective PD. For information on this refer to [this section of the PD docs](../pd-disaggregation/README.md#tuning-selective-pd).

## Verifying the installation

- Firstly, you should be able to list all helm releases installed into your chosen namespace:

```bash
helm list -n ${NAMESPACE}
NAME            NAMESPACE       REVISION    UPDATED                                 STATUS      CHART                       APP VERSION
llm-d-infpool   llm-d-wide-ep   1           2025-08-24 13:14:53.355639 -0700 PDT    deployed    inferencepool-v1.0          v0.3.0
```

- Out of the box with this example you should have the following resources (if using Kgateway):

```bash
kubectl get all -n ${NAMESPACE}
NAME                                                          READY   STATUS    RESTARTS   AGE
pod/infra-wide-ep-inference-gateway-74d5c66c86-h5mfn          1/1     Running   0          2m22s
pod/llm-d-infpool-epp-84dd98f75b-r6lvh                        1/1     Running   0          2m14s
pod/llm-d-wide-ep-0-decode-sg-0-decode-leader-cq8vh           2/2     Running   0          2m13s
pod/llm-d-wide-ep-0-decode-sg-0-decode-worker-2qk4l           2/2     Running   0          2m13s
pod/llm-d-wide-ep-0-decode-sg-0-decode-worker-b8cnq           2/2     Running   0          2m13s
pod/llm-d-wide-ep-0-decode-sg-0-decode-worker-fj7kw           2/2     Running   0          2m13s
pod/llm-d-wide-ep-0-decode-sg-0-decode-worker-gxl3n           2/2     Running   0          2m13s
pod/llm-d-wide-ep-0-decode-sg-0-decode-worker-m02zv           2/2     Running   0          2m13s
pod/llm-d-wide-ep-0-decode-sg-0-decode-worker-w8j2c           2/2     Running   0          2m13s
pod/llm-d-wide-ep-0-decode-sg-0-decode-worker-x1p9a           2/2     Running   0          2m13s
pod/llm-d-wide-ep-0-prefill-cckkt                             1/1     Running   0          2m13s


NAME                                              TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)                        AGE
service/infra-wide-ep-inference-gateway           ClusterIP      10.16.1.34      10.16.4.2     15021:30312/TCP,80:33662/TCP   2m22s
service/llm-d-infpool-epp                         ClusterIP      10.16.1.137     <none>        9002/TCP                       2d4h
service/llm-d-wide-ep-0                           ClusterIP      None            <none>        <none>                         2m13s

NAME                                                   READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/infra-wide-ep-inference-gateway        1/1     1            1           2m22s
deployment.apps/llm-d-infpool-epp                      1/1     1            1           2m14s

NAME                                                              DESIRED   CURRENT   READY   AGE
replicaset.apps/infra-wide-ep-inference-gateway-74d5c66c86        1         1         1       2m22s
replicaset.apps/llm-d-infpool-epp-55bb9857cf                      1         1         1       2m14s
```


**_NOTE:_** This assumes no other guide deployments in your given `${NAMESPACE}` and you have not changed the default release names via the `${RELEASE_NAME}` environment variable.

## Using the stack

For instructions on getting started making inference requests see [our docs](../../docs/getting-started-inferencing.md)

**_NOTE:_** This example particularly benefits from utilizing stern as described in the [getting-started-inferencing docs](../../docs/getting-started-inferencing.md#following-logs-for-requests), because while we only have 3 inferencing pods, it has 16 vllm servers or ranks.

**_NOTE:_** Compared to the other examples, this one takes anywhere between 7-10 minutes for the vllm API servers to startup so this might take longer before you can interact with this example.

## Cleanup

To remove the deployment:

```bash
# From guides/wide-ep
helm uninstall llm-d-infpool -n ${NAMESPACE}
kubectl delete -k ./manifests/modelserver/grove/ -n ${NAMESPACE}
kubectl delete -k ./manifests/gateway/<gke-l7-regional-external-managed|istio|kgateway|kgateway-openshift> -n ${NAMESPACE}
```

## Customization

For information on customizing a guide and tips to build your own, see [our docs](../../docs/customizing-a-guide.md)

