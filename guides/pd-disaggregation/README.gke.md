# PD-Disaggregation on GKE (DRA & RoCE)

This guide provides specific instructions for deploying P/D (Prefill/Decode) disaggregation on GKE using **Dynamic Resource Allocation (DRA)** for NVIDIA GPUs and RDMA (RoCE) networking. For the purposes of this guide, we will be using NVIDIA H200 GPUs on A3 Ultra.

**Note**: Follow the official GCP documentation for the latest updates and detailed instructions:
* [GKE AI Hypercomputer Custom Provisioning](https://docs.cloud.google.com/ai-hypercomputer/docs/create/gke-ai-hypercompute-custom)
* [Set up GPU Dynamic Resource Allocation (DRA)](https://docs.cloud.google.com/kubernetes-engine/docs/how-to/set-up-dra)
* [Allocate network resources by using GKE managed DRANET](https://docs.cloud.google.com/kubernetes-engine/docs/how-to/allocate-network-resources-dra#use-rdma-interfaces-gpu)

## Prerequisites

Set up the environment variables for your deployment:

```bash
export PROJECT="<your GCP project>"
export LOCATION="<your GCP location>"
export CLUSTER_NAME="<your cluster name>"
export NAMESPACE="llm-d-pd-disaggregation"
```

### 1. Create Cluster (Managed DRANET)

GKE network DRA (**DRANET**) supports managed networking out of the box, so you do not need to install custom networking drivers. However, you **must** enable **Dataplane V2** when creating your cluster:

```bash
gcloud container clusters create "${CLUSTER_NAME}" \
    --enable-dataplane-v2 \
    --location="${LOCATION}" \
    --project="${PROJECT}"
```

### 2. Create Node Pool (Enable DRANET & Disable Default GPU Driver)

GPU DRA is not yet fully managed by GKE. Therefore, you must disable the automated GPU driver and the default GPU device plugin via `--accelerator gpu-driver-version=disabled`. 

GKE managed **DRANET** is enabled by configuring `--accelerator-network-profile=auto` and adding the `cloud.google.com/gke-networking-dra-driver=true` node label:

```bash
gcloud beta container node-pools create a3u-dra-pool-1 \
  --project="${PROJECT}" \
  --location="${LOCATION}" \
  --cluster="${CLUSTER_NAME}" \
  --accelerator type=nvidia-h200-141gb,count=8,gpu-driver-version=disabled \
  --machine-type=a3-ultragpu-8g \
  --num-nodes=2 \
  --spot \
  --accelerator-network-profile=auto \
  --node-labels="cloud.google.com/gke-networking-dra-driver=true,goog-gke-accelerator-type=nvidia-h200-141gb,nvidia.com/gpu.present=true,cloud.google.com/gke-nvidia-gpu-dra-driver=true,gke-no-default-nvidia-gpu-device-plugin=true"
```

### 3. Install NVIDIA & GPU DRA Drivers

Since automated GPU driver management is disabled, you must install the NVIDIA Driver and the NVIDIA GPU DRA Driver manually.

#### 3.1 Install NVIDIA Driver

Apply the preloaded COS NVIDIA driver DaemonSet:

```bash
kubectl apply -f https://raw.githubusercontent.com/GoogleCloudPlatform/container-engine-accelerators/master/nvidia-driver-installer/cos/daemonset-preloaded.yaml
```

#### 3.2 Install NVIDIA GPU DRA Driver

Install the official NVIDIA DRA Driver:

```bash
helm install nvidia-dra-driver-gpu nvidia/nvidia-dra-driver-gpu \
    --version="25.8.0" --create-namespace --namespace=nvidia-dra-driver-gpu \
    --set nvidiaDriverRoot="/home/kubernetes/bin/nvidia/" \
    --set gpuResourcesEnabledOverride=true \
    --set resources.computeDomains.enabled=false \
    --set kubeletPlugin.priorityClassName="" \
    --set 'kubeletPlugin.tolerations[0].key=nvidia.com/gpu' \
    --set 'kubeletPlugin.tolerations[0].operator=Exists' \
    --set 'kubeletPlugin.tolerations[0].effect=NoSchedule'
```

#### 3.3 Verify DRA Device Readiness

Verify that the DRA driver pods are running and `ResourceSlices` are successfully populated with `gpu.nvidia.com` and `mrdma.google.com` devices:

```bash
kubectl get pods -n nvidia-dra-driver-gpu
```

```console
NAME                                         READY   STATUS    RESTARTS   AGE
nvidia-dra-driver-gpu-kubelet-plugin-52cdm   1/1     Running   0          46s
```

```bash
kubectl get resourceslices -o yaml
```

<details>
<summary><b>Click to view expected output</b></summary>

```yaml
apiVersion: v1
items:
- apiVersion: resource.k8s.io/v1
  kind: ResourceSlice
  metadata:
    name: gpu-slice-0
  spec:
    devices:
    - attributes:
        productName:
          string: NVIDIA H200
        resource.kubernetes.io/pcieRoot:
          string: pci0000:00
        type:
          string: gpu
      name: gpu-0
    driver: gpu.nvidia.com
    nodeName: a3u-dra-pool-1-node
- apiVersion: resource.k8s.io/v1
  kind: ResourceSlice
  metadata:
    name: nic-slice-0
  spec:
    devices:
    - attributes:
        resource.kubernetes.io/pcieRoot:
          string: pci0000:00
        type:
          string: nic
      name: nic-0
    driver: mrdma.google.com
    nodeName: a3u-dra-pool-1-node
```

</details>

## 1. Deploy the Model Server

Deploy vLLM with the GKE overlay. Note that the GKE overlay uses Dynamic Resource Allocation (`ResourceClaimTemplate`) to co-allocate GPUs and RDMA network interfaces aligned under the same PCIe root.

```bash
kubectl apply -n ${NAMESPACE} -k guides/pd-disaggregation/modelserver/gpu/vllm/gke/
```

## 2. Verification & Logs

Monitor pod scheduling and model startup:

```bash
kubectl get pods -n ${NAMESPACE} -w
kubectl logs -n ${NAMESPACE} -l llm-d.ai/role=decode -c modelserver --tail=100
```
