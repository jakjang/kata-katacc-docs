# Installing Pod Sandboxing on AKS

*Note, there are steps to enable the aks-preview extension and registering the KataVMIsolationPreview feature flag in the current preview iteration. 
Come GA, these steps should no longer be necessary. As a result, they're being left out of this guide. Please refer to the existing [guide](https://learn.microsoft.com/en-us/azure/aks/use-pod-sandboxing)
for more details on the aforementioned steps.

## Limitations 
- Kata Containers may not reach IOPS performance limits that traditional containers can reach on Azure Files and high performance local SSDs.
- [Microsoft Defender for Containers](https://learn.microsoft.com/en-us/azure/defender-for-cloud/defender-for-containers-introduction) doesn't support assessing Kata runtime pods.
- [Kata](https://github.com/kata-containers/kata-containers/blob/main/docs/Limitations.md#host-network) host-network isn't supported.

## Create a resource group

Call the [az group create](https://learn.microsoft.com/en-us/cli/azure/group?view=azure-cli-latest#az-group-create) command
```azurecli-interactive
az group create --name myResourceGroup --location eastus
```

## Deploying a new cluster
1. Create an AKS cluster using the [az aks create](https://learn.microsoft.com/en-us/cli/azure/aks#az-aks-create) command. Specify the following parameters:
   - --workload-runtime: *KataMshvVmIsolation* in order to enable Pod Sandboxing on the node pool.
   - --os-sku: *AzureLinux*. Only the Azure Linux os-sku supports this feature.
   - --node-vm-size: Ensure the Azure VM size you select is a generation 2 VM and supports nested virtualization. [Dsv3](https://learn.microsoft.com/en-us/azure/virtual-machines/dv3-dsv3-series#dsv3-series) series VMs are an example.
The following shows an example:
```azurecli-interactive
az aks create \
--name myAKSCluster \
--resource-group myResourceGroup \
--os-sku AzureLinux \
--workload-runtime KataMshvVmIsolation \ 
--node-vm-size Standard_D4s_v3 \
--node-count 1 \
--generate ssh keys
```
1. Once your cluster is ready, you should get some JSON-formatted information about your cluster. Run the following command to get access crednetials for the cluster:
```azurecli-interactive
az aks get-credentials --resource-group myResourceGroup --name myAKSCluster
```

## Deploying to an existing cluster
If you already have an existing AKS cluster and Azure resource group, you can deploy a new node pool directly to your existing cluster.
1. Add a node pool to your AKS cluster using the [az aks nodepool add](https://learn.microsoft.com/en-us/cli/azure/aks/nodepool#az-aks-nodepool-add) command. Specify the following parameters: 
   - --workload-runtime: *KataMshvVmIsolation* to enable the Pod Sandboxing feature on the node pool.
   - --os-sku: *AzureLinux*. Only the Azure Linux os-sku supports this feature.
   - --node-vm-size: Ensure the Azure VM size you select is a generation 2 VM and supports nested virtualization. [Dsv3](https://learn.microsoft.com/en-us/azure/virtual-machines/dv3-dsv3-series#dsv3-series) series VMs are an example.
   - --resource-group: Enter the name of your existing resource group.
   - --cluster-name: Use your existing AKS cluster name.
   - --name: Enter a unique name for your new node pool.
The following shows an example:
```azurecli-interactive
az aks nodepool add \
--cluster-name myAKSCluster \
--resource-group myResourceGroup \
--name nodepool2 \
--os-sku AzureLinux \
--workload-runtime KataMshvVmIsolation \
--node-vm-size Standard_D4s_v3
```
1. Run the [az aks update](https://learn.microsoft.com/en-us/cli/azure/aks#az-aks-update) command to enable pod sandboxing on the cluster.

## Deploying applications

With pod sandboxing enabled, applications you deploy should be considered in the scope of being **trusted** or **untrusted**.
- Trusted applications can be deployed as normal. Using your existing pod manifests, you can deploy you applications as is. They'll be deployed like any normal pod on your
cluster.
- Untrusted applications will be deployed in children Kata Virtual Machines (KVMs), which are their own trust boundary, isolated from the host kernel. Each untrusted application you deploy will be created
in their own KVM.
   - Of note, inside your pod manifest `spec:`, adding the line `runtimeClassName: kata-mshv-vm-isolation` will denote an untrusted application.

## Experimenting 
If you would like to test for yourself, here are some pods to test out the difference.

A trusted pod manifest will look like any normal one. Take this example trusted pod. Please save it as `trusted-app.yaml`
```yml
kind: Pod
apiVersion: v1
metadata:
  name: trusted
spec:
  containers:
  - name: trusted
    image: mcr.microsoft.com/aks/fundamental/base-ubuntu:v0.0.11
    command: ["/bin/sh", "-ec", "while :; do echo '.'; sleep 5 ; done"]
```

An untrusted pod manifest, as mentioned above, will have an additional `runtimeClassName`. Take this example untrusted pod. Please save it as `untrusted-app.yaml`
```yml
kind: Pod
apiVersion: v1
metadata:
  name: untrusted
spec:
  runtimeClassName: kata-mshv-vm-isolation
  containers:
  - name: untrusted
    image: mcr.microsoft.com/aks/fundamental/base-ubuntu:v0.0.11
    command: ["/bin/sh", "-ec", "while :; do echo '.'; sleep 5 ; done"]
```
Deploy your Kubernetes pods by running the [kubectl apply](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#apply) command.
For your trusted app:
```bash
kubectl apply -f trusted-app.yaml
```
and for your untrusted app:
```bash
kubectl apply -f untrusted-app.yaml
```
Now to verify kernel isolation between these pods, the [kubectl exec](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#exec) command lets you start shell sessions into each pod.
```bash
kubectl exec -it untrusted -- /bin/bash
```
```bash
uname -r
```
Should show you the kernel version of the untrusted pod.

You can compare it to the kernel version of your trusted pod. You should notice a difference.
```bash
kubectl exec -it trusted -- /bin/bash
```
```bash
uname -r
```
