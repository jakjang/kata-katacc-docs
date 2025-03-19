# Setup prerequisites

## Install the aks-preview Azure CLI extension 

To install the aks-preview extension, run the following command:

```azurecli-interactive
az extension add --name aks-preview
```

### Install the confcom Azure CLI extension

To install the confcom extension, run the following command:

```azurecli-interactive
az extension add --name confcom
```

Run the following command to update to the latest version of the extension:

```azurecli-interactive
az extension update --name confcom
```

### Register the KataCcIsolationPreview feature flag

Register the `KataCcIsolationPreview` feature flag by using the [az feature register][az-feature-register] command, as shown in the following example:

```azurecli-interactive
az feature register --namespace "Microsoft.ContainerService" --name "KataCcIsolationPreview"
```

It takes a few minutes for the status to show *Registered*. Verify the registration status by using the [az feature show][az-feature-show] command:

```azurecli-interactive
az feature show --namespace "Microsoft.ContainerService" --name "KataCcIsolationPreview"
```

When the status reflects *Registered*, refresh the registration of the *Microsoft.ContainerService* resource provider by using the [az provider register][az-provider-register] command:

```azurecli-interactive
az provider register --namespace "Microsoft.ContainerService"
```

# Set up your environmental variables
As you go through this exercise, you will find yourself using the same values in many spots. Configuring environmental variables can save you some time. There are also resources that require unique names; functions alongside the environmental variable helps simplify this. 

First, ensure you are in the proper subscription.
```azurecli-interactive
az account set --subscription <your subscription ID>
```

*Note: feel free to name the variables whatever you want. Please make sure to make the names memorable, as you will call on them later on.*
```azurecli-interactive
export RANDOM_ID="$(openssl rand -hex 3)"
export RESOURCE_GROUP="myResourceGroup$RANDOM_ID"
export LOCATION="eastus"
export CLUSTER_NAME="myAKSCluster$RANDOM_ID"
export SUBSCRIPTION="$(az account show --query id --output tsv)"
export USER_ASSIGNED_IDENTITY_NAME="myIdentity$RANDOM_ID"
export SERVICE_ACCOUNT_NAMESPACE="kafka"
export SERVICE_ACCOUNT_NAME="workload-identity-sa$RANDOM_ID"
export FEDERATED_IDENTITY_CREDENTIAL_NAME="myFedIdentity$RANDOM_ID"
export KEY_VAULT_NAME="keyvault$RANDOM_ID"
```
# Create a resource group

asdfasf

# Deploy a new AKS cluster
1. Create an AKS cluster using the [az aks create](https://learn.microsoft.com/en-us/cli/azure/aks#az_aks_create) command, specifying:

   * **--os-sku**: *AzureLinux*. Only the Azure Linux os-sku supports this feature in this preview release.
   * **--node-vm-size**: Any Azure VM size that supports AMD SEV-SNP protected child VMs works. For example, [Standard_DC8as_cc_v5][DC8as-series] VMs.
   * **--enable-workload-identity**: Enables creating a Microsoft Entra Workload ID enabling pods to use a Kubernetes identity.
   * **--enable-oidc-issuer**: Enables OpenID Connect (OIDC) Issuer. It allows a Microsoft Entra ID or other cloud provider identity and access management platform the ability to discover the API server's public signing keys.
   * **--workload-runtime**: Specify *KataCcIsolation* to enable the Confidential Containers feature on the node pool.

    ```azurecli-interactive
   az aks create \
    --resource-group "${RESOURCE_GROUP}" \
    --name "${CLUSTER_NAME}" \
    --kubernetes-version <1.25.0 and above> \
    --os-sku AzureLinux \
    --node-vm-size Standard_DC8as_cc_v5 \
    --workload-runtime KataCcIsolation \
    --node-count 1 \
    --enable-oidc-issuer \
    --enable-workload-identity \
    --generate-ssh-keys
   ```
   After a few minutes, the command completes and returns JSON-formatted information about the cluster.

1. When the cluster is ready, get the cluster credentials using the [az aks get-credentials][az-aks-get-credentials] command.

    ```azurecli-interactive
    az aks get-credentials --resource-group "${RESOURCE_GROUP}" --name ${CLUSTER_NAME}"
    ```
3. Create a namespace to run your apps and kafka cluster to run out of. 

 ```azurecli-interactive
    kubectl create namespace kafka
 ```

# Configure workload identity 
Confidential Containers uses Azure Key Vault (AKV) and RBAC for public/private key creation and release. Before you configure access to AKV and deploy your applications, you need to configure your workload identity.

## Retrieve the OIDC Issuer URL

Get the OIDC issuer URL and save it to an environmental variable.
   
   ```azurecli-interactive
   export AKS_OIDC_ISSUER="$(az aks show --name "${CLUSTER_NAME}" \
    --resource-group "${RESOURCE_GROUP}" \
    --query "oidcIssuerProfile.issuerUrl" \
    --output tsv)"
   ```

## Create a managed identity

1. Create your managed identity.
   
   ```azurecli-interactive
    export SUBSCRIPTION="$(az account show --query id --output tsv)"
    export USER_ASSIGNED_IDENTITY_NAME="myIdentity$RANDOM_ID"
    az identity create \
        --name "${USER_ASSIGNED_IDENTITY_NAME}" \
        --resource-group "${RESOURCE_GROUP}" \
        --location "${LOCATION}" \
        --subscription "${SUBSCRIPTION}"
   ```
1. Save the managed identity's client ID as a variable.
   ```azurecli-interactive
   export USER_ASSIGNED_CLIENT_ID="$(az identity show \
    --resource-group "${RESOURCE_GROUP}" \
    --name "${USER_ASSIGNED_IDENTITY_NAME}" \
    --query 'clientId' \
    --output tsv)"
   ```

## Create a Kubernetes service account

```azurecli-interactive
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  annotations:
    azure.workload.identity/client-id: "${USER_ASSIGNED_CLIENT_ID}"
  name: "${SERVICE_ACCOUNT_NAME}"
  namespace: "${SERVICE_ACCOUNT_NAMESPACE}"
EOF
```
If successful, you should see an output similar to the one below:
```output
serviceaccount/workload-identity-saXXXXX created
```

## Create the federated identity credential
Create the FIC between the managed identity, service account issuer, and the subject. 
```azurecli-interactive
az identity federated-credential create \
    --name ${FEDERATED_IDENTITY_CREDENTIAL_NAME} \
    --identity-name "${USER_ASSIGNED_IDENTITY_NAME}" \
    --resource-group "${RESOURCE_GROUP}" \
    --issuer "${AKS_OIDC_ISSUER}" \
    --subject system:serviceaccount:"${SERVICE_ACCOUNT_NAMESPACE}":"${SERVICE_ACCOUNT_NAME}" \
    --audience api://AzureADTokenExchange
```

# Create an Azure Key Vault
When you make your key vault, ensure you use the *Premium* SKU to ensure support for HSM keys.
```azurecli-interactive
az keyvault create --name "$(KEY_VAULT_NAME}" --resource-group "${RESOUCE_GROUP}" --sku "Premium"
```

## Configure access in your key vault
Grant both yourself and the managed identity you created earlier access to the key vault by giving both identities the **Key Vault Crypto Officer** and **Key Vault Crypto User** Azure RBAC roles.

# Set up Azure Attestation.
## Set up your prerequisites
1. Install the extension
   ```azurecli-interactive
   az extension add --name attestation
   ```
1. Register the Microsoft.Attestation resource provider in your subscription
   ```azurecli-interactive
   az provider register --name Microsoft.Attestation
   ```
## Create an attestation provider.
1. Set up an environmental variable for your attestation provider's name
   ```azurecli-interactive
   export ATTEST_NAME="myattest$RANDOM_ID"
   ```
1. Run the [az attestation create](https://learn.microsoft.com/en-us/cli/azure/attestation#az-attestation-create) command to create an attestation provider
   ```azurecli-interactive
   az attestation create --name "${ATTEST_NAME}" --resource-group "${RESOURCE_GROUP}" --location "${LOCATION}"
   ```
