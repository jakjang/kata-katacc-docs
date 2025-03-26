*Only instructions and minimal additional text is included to streamline testing/creation. More details can/should be included if this ever gets expanded to be user facing.*

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

# Set up your AKS Resources

## Environmental variables 
As you go through this exercise, you will find yourself using the same values in many commands. Configuring environmental variables can save you some time.
*Note: feel free to name the variables whatever you want. Please make sure to make the names memorable, as you will call on them later on.*
```azurecli-interactive
export RANDOM_ID="$(openssl rand -hex 3)"
export RESOURCE_GROUP="myResourceGroup$RANDOM_ID"
export LOCATION="eastus"
export CLUSTER_NAME="myAKSCluster$RANDOM_ID"
export SUBSCRIPTION="$(az account show --query id --output tsv)"
```
## Set up your resource group
*This step assumes you don't already have a resource group and/or cluster setup. If you do, head over to "Deploy to an existing cluster"*

Call the [az group create](https://learn.microsoft.com/en-us/cli/azure/group?view=azure-cli-latest#az-group-create) command
```azurecli-interactive
az group create --name "${RESOURCE_GROUP}" --location "${LOCATION}"
```

## Deploy a new AKS cluster
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
    az aks
    ```

## Deploying to an existing cluster
If you are deploying to an exsting cluster, enable Confidental Containers by creating a node pool to host it.

1. Add a node pool using the [az aks nodepool add](https://learn.microsoft.com/en-us/cli/azure/aks/nodepool#az_aks_nodepool_add) command. Ensure you include the following parameters:
      * **--workload-runtime**: Specify *KataCcIsolation* to enable Confidential Containers on the node pool.
      * **--os-sku**: *AzureLinux*. Only the Azure Linux os-sku supports this feature.
      * **--node-vm-size**: Any Azure VM size that supports AMD SEV-SNP protected child VMs and nested virtualization will work. [Standard_DC8as_cc_v5](https://learn.microsoft.com/en-us/azure/virtual-machines/dcasccv5-dcadsccv5-series) VMs are one such example.

   ```azurecli-interactive
   az aks nodepool add \
   --resource-group ${RESOURCE_GROUP} \
   --cluster-name ${CLUSTER_NAME} \
   --name nodepool2 \
   --node-count 2 \
   --os-sku AzureLinux \
   --node-vm-size Standard_DC8as_cc_v5 \
   --workload-runtime KataCcIsolation
   ```
   After a few minutes, the command completes and returns JSON-formatted information about the cluster.

1. Run  the [az aks update](https://learn.microsoft.com/en-us/cli/azure/aks#az_aks_update) command to eanble Confidential Containers on the cluster

   ```azurecli-interactive
   az aks update --name ${CLUSTER_NAME} --resource-group ${RESOURCE_GROUP}
   ```

1. When the cluster is ready, get the cluster credentials using the [az aks get-credentials][az-aks-get-credentials] command.

    ```azurecli-interactive
    az aks get-credentials --resource-group ${RESOURCE_GROUP} --name {CLUSTER_NAME}
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
export KEYVAULT_NAME="keyvault$RANDOM_ID"
```
# Create a resource group

Call the [az group create](https://learn.microsoft.com/en-us/cli/azure/group?view=azure-cli-latest#az-group-create) command
```azurecli-interactive
az group create --name "${RESOURCE_GROUP}" --location "${LOCATION}"
```


1. Create a namespace to run your apps and kafka cluster to run out of. 

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
1. Save your managed identity's principal ID as a variable
   ```azurecli-interactive
   export MANAGED_ID=$(az identity show --resource-group ${RESOURCE_GROUP} --name ${USER_ASSIGNED_IDENTITY_NAME} --query principalId --output tsv)
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
az keyvault create --name "${KEY_VAULT_NAME}" --resource-group "${RESOURCE_GROUP}" --sku "Premium"
```

## Configure access in your key vault
Grant both yourself and the managed identity you created earlier access to the key vault by giving both identities the **Key Vault Crypto Officer** and **Key Vault Crypto User** Azure RBAC roles.
1. Assign the Crypto Officer role
   ```azurecli-interactive
   export KEYVAULT_RESOURCE_ID=$(az keyvault show --resource-group "${KEYVAULT_RESOURCE_GROUP}" \
    --name "${KEYVAULT_NAME}" \
    --query id \
    --output tsv)

   export CALLER_OBJECT_ID=$(az ad signed-in-user show --query id -o tsv)

   az role assignment create --assignee "${CALLER_OBJECT_ID}" \
       --role "Key Vault Crypto Officer" \
       --scope "${KEYVAULT_RESOURCE_ID}"
   ```
   And to your managed ID
   ```azurecli-interactive
    az role assignment create --assignee "${MANAGED_ID}" \
       --role "Key Vault Crypto Officer" \
       --scope "${KEYVAULT_RESOURCE_ID}"
   ```
1. Assign the Crypto User role
   First to your account
   ```azurecli-interactive
      az role assignment create --assignee "${CALLER_OBJECT_ID}" \
       --role "Key Vault Crypto User" \
       --scope "${KEYVAULT_RESOURCE_ID}"
   ```
   Then to your managed ID
      ```azurecli-interactive
      az role assignment create --assignee "${MANAGED_ID}" \
       --role "Key Vault Crypto User" \
       --scope "${KEYVAULT_RESOURCE_ID}"
   ```

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
1. Set your `MAA_ENDPOINT` environmental variable with the FQDN of Attest URI
   ```azurecli-interactive
   export MAA_ENDPOINT="$(az attestation show --name ${ATTEST_NAME} --resource-group ${RESOURCE_GROUP} --query 'attestUri' -o tsv | cut -c 9-)"
   ```
   *You can double check if FQDN of the Attest URI is in the proper format by running `echo $MAA_ENDPOINT`

# Setup your kafka cluster
1. Install the kafka cluster in the kafka namespace
   ```azurecli-interactive
   kubectl create -f 'https://strimzi.io/install/latest?namespace=kafka' -n kafka
   ```
1. Setup the kafka cluster by running the pod manifest
   ```bash
   kubectl apply -f https://strimzi.io/examples/latest/kafka/kafka-persistent-single.yaml -n kafka
   ```

# Prepare your keys
Prepare the RSA encryption/decryption key using this [bash script](https://github.com/microsoft/confidential-container-demos/raw/main/kafka/setup-key.sh). For now, save the file as `setup-key.sh`

# Set up your consumer and producer pod manifests
1. Copy and save the following as `consumer.yaml`
   ```yml
    apiVersion: v1
    kind: Pod
    metadata:
      name: kafka-golang-consumer
      namespace: kafka
      labels:
        azure.workload.identity/use: "true"
        app.kubernetes.io/name: kafka-golang-consumer
    spec:
      serviceAccountName: workload-identity-sa
      runtimeClassName: kata-cc-isolation
      containers:
        - image: "mcr.microsoft.com/aci/skr:2.7"
          imagePullPolicy: Always
          name: skr
          env:
            - name: SkrSideCarArgs
              value: ewogICAgImNlcnRjYWNoZSI6IHsKCQkiZW5kcG9pbnRfdHlwZSI6ICJMb2NhbFRISU0iLAoJCSJlbmRwb2ludCI6ICIxNjkuMjU0LjE2OS4yNTQvbWV0YWRhdGEvVEhJTS9hbWQvY2VydGlmaWNhdGlvbiIKCX0gIAp9
          command:
            - /bin/skr
          volumeMounts:
            - mountPath: /opt/confidential-containers/share/kata-containers/reference-info-base64
              name: endor-loc
        - image: "mcr.microsoft.com/acc/samples/kafka/consumer:1.0"
          imagePullPolicy: Always
          name: kafka-golang-consumer
          env:
            - name: SkrClientKID
              value: kafka-encryption-demo
            - name: SkrClientMAAEndpoint
              value: sharedeus2.eus2.test.attest.azure.net
            - name: SkrClientAKVEndpoint
              value: myKeyVault.vault.azure.net
            - name: TOPIC
              value: kafka-demo-topic
          command:
            - /consume
          ports:
            - containerPort: 3333
              name: kafka-consumer
          resources:
            limits:
              memory: 1Gi
              cpu: 200m
      volumes:
        - name: endor-loc
          hostPath:
            path: /opt/confidential-containers/share/kata-containers/reference-info-base64
    ---
    apiVersion: v1
    kind: Service
    metadata:
      name: consumer
      namespace: kafka
    spec:
      type: LoadBalancer
      selector:
        app.kubernetes.io/name: kafka-golang-consumer
      ports:
        - protocol: TCP
          port: 80
          targetPort: kafka-consumer
   ```
   Ensure you update:
      - `SkrClientAKVEndpoint` value to match the URL of your Azure Key Vault, excluding the protocol value `https://`
      - `SkrClientMAAEndpoint` value with the value of your MAA_ENDPOINT
      - `serviceAccountName` value with your SERVICE_ACCOUNT_NAME's name

1. Generate the security policy for the consumer YAML manifest and store the hash in environmental variable `WORKLOAD_MEASUREMENT`
   ```bash
   export WORKLOAD_MEASUREMENT=$(az confcom katapolicygen -y consumer.yaml --print-policy | base64 -d | sha256sum | cut -d' ' -f1)
   ```

1. Generate an RSA asymmetric key pair (public/private keys) by running the `setup-key.sh` script with the following command. Replace the AKV URL with your own unique URL, without the protocol value `https://`
   ```bash
   export MANAGED_IDENTITY=${USER_ASSIGNED_CLIENT_ID}
   bash setup-key.sh "kafka-encryption-demo" <Azure Key Vault URL>
   ```
>**Note** Once the key is generated, the public key will be saved as `kafka-encryption-demo-pub.pem`. You can get it's value by running
   ```bash
   cat kafka-encryption-demo-pub.pem 
   ```

1. Copy and save the following as `producer.yaml`.
   ```yml
    apiVersion: v1
    kind: Pod
    metadata:
      name: kafka-producer
      namespace: kafka
    spec:
      containers:
        - image: "mcr.microsoft.com/acc/samples/kafka/producer:1.0"
          name: kafka-producer
          command:
            - /produce
          env:
            - name: TOPIC
              value: kafka-demo-topic
            - name: MSG
              value: "Azure Confidential Computing"
            - name: PUBKEY
              value: |-
                -----BEGIN PUBLIC KEY-----
                MIIBojAN***AE=
                -----END PUBLIC KEY-----
          resources:
            limits:
              memory: 1Gi
              cpu: 200m
   ```
   >**Note**: Update the value which begin with `-----BEGIN PUBLIC KEY-----` and ends with `-----END PUBLIC KEY-----` with the public key from kafka-encryption-demo-pub.pem.

# Test Confidential Containers

1. Deploy the `consumer.yaml` and `producer.yaml` you created earlier.
   ```bash
   kubectl apply -f consumer.yaml
  ```bash
   kubectl apply -f producer.yaml
   ```
1. Get the IP address of the web service
   ```bash
   kubectl get svc consumer -n kafka
   ```
1. Copy/paste the external IP address into a web browser.
   If everything is working correctly, the output should look like
   ```output
   Welcome to Confidential Containers on AKS!
   Encrypted Kafka Message:
   Msg 1: Azure Confidential Computing
   ```

# Further testing
You can also try to run the consumer pod as a regular K8s pod by removing the `skr container` and `kata-cc` runtime spec from the pod manifest to observe what happens. 
