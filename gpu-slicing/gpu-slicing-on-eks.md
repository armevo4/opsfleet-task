### What is GPU Slicing?
GPU slicing allows users to share a single GPU across multiple containers or workloads by dividing it into smaller segments (slices). This feature is essential when workloads don’t need an entire GPU, allowing you to maximize GPU utilization and reduce costs. In AWS, this can be done using NVIDIA’s Multi-Instance GPU (MIG) feature, which enables partitioning of NVIDIA A100 GPUs.

### How GPU Slicing Works on EKS
GPU slicing is possible with the NVIDIA A100 GPUs, which support the MIG feature. MIG allows partitioning of the GPU into smaller, isolated instances that can run independent workloads. This is especially beneficial for Kubernetes clusters like EKS because it allows more efficient utilization of GPU resources and reduces the cost of running AI/ML workloads.

Here’s how you can enable GPU slicing on EKS clusters:

### Step 1: Update Your EKS Cluster
Ensure your EKS cluster is running the latest Kubernetes version, ideally 1.23 or later, which includes support for new GPU features. You can check your version with:

```bash
kubectl version
```

If an upgrade is needed, refer to AWS documentation on upgrading an EKS cluster.

### Step 2: Deploy NVIDIA GPU Operator
NVIDIA provides the GPU Operator to manage GPU drivers and libraries. You'll need to install this operator to enable MIG. Here's how you can deploy it:

1. Add the NVIDIA Helm repository:
```bash
helm repo add nvidia https://nvidia.github.io/gpu-operator
helm repo update
```
2. Install the GPU Operator:
```bash
helm install --wait --generate-name \
    nvidia/gpu-operator \
    --create-namespace \
    --namespace gpu-operator
```
3. Verify the installation:

### Step 3: Enable MIG on A100 GPUs
To enable GPU slicing, you need to configure MIG profiles for NVIDIA A100 GPUs. Here’s how:

1. Label your nodes with GPU instances: Once the NVIDIA GPU operator is deployed, label the GPU nodes for MIG:

```bash
kubectl label node <node-name> nvidia.com/mig.config=all-1g.5gb
```
The all-1g.5gb config creates 7 instances of 1GB each on an A100 GPU. You can adjust the partitioning according to workload requirements.

2. Configure MIG profiles: MIG profiles are configured by modifying the GPU operator’s configuration. You’ll need to enable and specify the MIG strategy in the GPU operator configuration file:

```yaml
apiVersion: nvidia.com/v1
kind: ClusterPolicy
metadata:
  name: cluster-policy
spec:
  mig:
    strategy: single
    devices:
      - gpu: all
        profiles: "all-1g.5gb"
```

3. Restart GPU Nodes: Once MIG is configured, you may need to restart the GPU nodes for the configuration to take effect.

### Step 4: Modify Workload Requests to Use Slices
Now that GPU slicing is enabled, your AI workloads need to request specific MIG profiles. You can modify the Kubernetes deployment YAML to specify the desired GPU slice:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: ai-workload
spec:
  containers:
  - name: gpu-container
    image: your-ai-model-image
    resources:
      limits:
        nvidia.com/gpu: 1  # Request a MIG slice (instead of full GPU)
      requests:
        memory: "2Gi"
        cpu: "1000m"
```

### Step 5: Enable Karpenter Autoscaling with GPU Slicing
Karpenter is an autoscaler for Kubernetes that dynamically provisions nodes based on workload demands. To leverage GPU slicing with Karpenter:

1. Install Karpenter (if not already installed): Follow Karpenter’s installation guide for EKS: Karpenter Installation

2. Configure Instance Types with GPU Support: When defining provisioners for Karpenter, ensure that you include instance types with A100 GPUs:

```yaml
apiVersion: karpenter.sh/v1alpha5
kind: Provisioner
metadata:
  name: gpu-provisioner
spec:
  provider:
    instanceProfile: KarpenterNodeInstanceProfile
  requirements:
    - key: "nvidia.com/gpu"
      operator: In
      values: ["true"]
  instanceTypes:
    - p4d.24xlarge
    - g5.24xlarge

```

3. Set Up Custom Resource Definitions (CRDs) for GPUs: Karpenter allows dynamic scaling of GPU nodes by integrating with the node lifecycle. When workloads request GPU resources, Karpenter provisions nodes accordingly, considering the GPU slicing.

4. Tune Autoscaling: You can further tune autoscaling behavior based on your workload and MIG usage by adjusting Karpenter’s settings to ensure it provisions the right number of GPU nodes with MIG profiles.

### Conclusion
GPU slicing on EKS using NVIDIA MIG is a powerful way to optimize GPU resource usage, especially for AI/ML workloads that don't require the full capacity of a GPU. By enabling MIG and integrating with Karpenter autoscaler, your client can optimize GPU costs effectively.