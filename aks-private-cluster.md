# Create AKS Private Clusters

> Estimated Duration: 60 minutes

## Introduction

This challenge will cover deployment of a private AKS cluster fully integrated in a Virtual Network as well as deploying the sample app and configuring ingress. Deploying a private AKS cluster is a complex task. It requires multiple Azure resources to be deployed and configured BEFORE deploying AKS, including a VM jumpbox with Azure CLI & kubectl CLI installed to access the private AKS cluster.

The script blocks below in the sections below demonstrate how to complete all steps of this challenge to deploy a private AKS cluster. The script blocks use the Azure CLI to deploy and configure the Azure resources.

You need to fulfill these [requirements](environment-setup.md) to complete this challenge

## Success Criteria

- Verify the application is reachable over the ingress controller

## Learning Resources

These docs will help you achieving these objectives:

- [Create a private AKS cluster](https://learn.microsoft.com/en-us/azure/aks/private-clusters)
- [Sign in to a Linux virtual machine in Azure by using Microsoft Entra ID and OpenSSH](https://learn.microsoft.com/en-us/entra/identity/devices/howto-vm-sign-in-azure-ad-linux)

## Create resource group and virtual network

Export environment variables required for resource group:

```bash
LOCATION=eastus2
RG=rg-aks-workshop-private
PRIVATE_AKS=aks-private
```

Login to Azure and create resource group:

```bash
az login
az group create --name $RG --location $LOCATION
```

Export variables for the virtual network for the AKS cluster:

```bash
VNET_NAME=aks-vnet
VNET_PREFIX=10.13.0.0/16
AKS_SUBNET_NAME=aks-subnet
AKS_SUBNET_PREFIX=10.13.76.0/24
```

Create the virtual network for the AKS cluster:

```bash
az network vnet create -g $RG -n $VNET_NAME --address-prefix $VNET_PREFIX -l $LOCATION
az network vnet subnet create -g $RG -n $AKS_SUBNET_NAME --vnet-name $VNET_NAME --address-prefix $AKS_SUBNET_PREFIX
AKS_SUBNET_ID=$(az network vnet subnet show -n $AKS_SUBNET_NAME --vnet-name $VNET_NAME -g $RG --query id -o tsv)
```

## Create Private AKS Cluster

> **NOTE:** if using MinGW client such as git-bash in Windows, export this variable to avoid converting variables with a leading `/` into system paths:

```bash
export MSYS_NO_PATHCONV=1
```

Create Private AKS cluster with managed identity enabled:

```bash
az aks create -g $RG -n $PRIVATE_AKS -l $LOCATION \
    --generate-ssh-keys --enable-private-cluster \
    --vnet-subnet-id $AKS_SUBNET_ID \
    --network-plugin azure \
    --enable-managed-identity
```

Cluster creation will take a few minutes. Once created, attempt to connect and you will notice it will fail due to being a private cluster:

```bash
# Cluster-info
az aks get-credentials -n $PRIVATE_AKS -g $RG --overwrite
kubectl get node
```

To validate connectivity you can navigate to the created aks cluster resource in Azure Portal and clicking on [**Run command**](https://learn.microsoft.com/en-us/azure/aks/access-private-cluster?source=recommendations&tabs=azure-cli#run-commands-on-your-aks-cluster) under **Kubernetes Resources**

## Connect to Private Cluster

### Option 1: Using aks command invoke

Using `az aks command invoke` you can remotely invoke `kubectl` commands. To proceed with this option simply create this alias:

```bash
alias remote="az aks command invoke -g $RG -n $PRIVATE_AKS -c "
```

Connect to private cluster:

```bash
az aks get-credentials -n $PRIVATE_AKS -g $RG
```

Deploy sample application:

```bash
remote "kubectl create namespace helloworld"
remote "kubectl apply -f aks-helloworld-basic.yaml -n helloworld" --file manifests/aks-helloworld-basic.yaml
```

### Option 2: Create Jumpbox VM

In this option you will create a Jumpbox VM to connect to the Private AKS cluster. You will create a VM in the same vnet and install kubectl to have access to the API.

Export environment variables:

```bash
# Variables
VM_NAME=vm-jumpbox
VM_SKU=Standard_B2ms
VM_SUBNET_NAME=vm-subnet
VM_SUBNET_PREFIX=10.13.2.0/24
```

Create Subnet for the jumpbox:

```bash
az network vnet subnet create -n $VM_SUBNET_NAME --vnet-name $VNET_NAME -g "$RG" --address-prefixes $VM_SUBNET_PREFIX
```

Create jumpbox VM:

```bash
az vm create -n $VM_NAME -g $RG --image Ubuntu2204 --size $VM_SKU --generate-ssh-keys \
  --vnet-name $VNET_NAME --subnet $VM_SUBNET_NAME \
  --assign-identity --admin-username azureuser
```

Enable the Microsoft Entra login VM extension:

```bash
az vm extension set --publisher Microsoft.Azure.ActiveDirectory \
    --name AADSSHLoginForLinux -g $RG --vm-name $VM_NAME
```

Get username and Resource Group ID needed to grant VM admin login to your Entra login:

```bash
USERNAME=$(az account show --query user.name --output tsv)
RG_ID=$(az group show --resource-group $RG --query id -o tsv)
```

Next assign VM Admin Login role to your username:

```bash
az role assignment create --role "Virtual Machine Administrator Login" --assignee $USERNAME --scope $RG_ID
```

<details>
  <summary>This is only needed if your Entra domain and login username do not match (click to expand/collapse)</summary>
  Use this command to assigne role to User ID:

    ```bash
    USERID=$(az ad user list --filter "mail eq '$USERNAME'" --query [0].id -o tsv)
    az role assignment create --role "Virtual Machine Administrator Login" --assignee-object-id $USERID --scope $RG_ID
    ```
</details>

Export SSH configuration:

```bash
az ssh config --file ~/.ssh/config -n $VM_NAME -g $RG
```

<details>
  <summary>This section is only needed if your subscription does not allow you to log in via cli using device code (click to expand/collapse)</summary>
  Create managed identity and assign Contributor role to be able to login to Azure from jumpbox :

    ```bash
    VM_IDENTITY_NAME=${VM_NAME}-identity
    az identity create -g $RG -n $VM_IDENTITY_NAME
    az vm identity assign -n $VM_NAME -g $RG --identities $VM_IDENTITY_NAME
    VM_IDENTITY_PRINCIPALID=$(az identity show -n $VM_IDENTITY_NAME -g $RG --query principalId -o tsv)
    VM_IDENTITY_ID=$(az identity show -n $VM_IDENTITY_NAME -g $RG --query id -o tsv)
    az role assignment create --assignee $VM_IDENTITY_PRINCIPALID --role Contributor --scope $RG_ID
    ```

  Assign "Kubernetes Cluster Admin" role to the new VM identity so that you can connect to the AKS cluster from the Jumbbox:

    ```bash
    AKS_ID=$(az aks show --resource-group $RG --name $PRIVATE_AKS --query id --output tsv)
    az role assignment create --assignee $VM_IDENTITY_PRINCIPALID --role "Azure Kubernetes Service RBAC Cluster Admin" --scope $AKS_ID
    ```

  Output value of `VM_IDENTITY_ID` and copy it so that it can be pasted when connected to jumpbox

    ```bash
    echo $VM_IDENTITY_ID
    ```
</details>

Create remote alias to SSH into jumpbox using:

```bash
alias remote="az ssh vm -n $VM_NAME -g $RG"
```

Run the following commands to install kubectl in Jumpbox VM:

```bash
remote "curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash"
remote "sudo az aks install-cli"
remote "az login"
```

Log into Azure. **NOTE:** if your subscription does not support device login, then use this command to login with the "VM_IDENTITY_ID" created: `az login --identity -u $VM_IDENTITY_ID`

```bash
remote "az login"
```

Resume logging into AKS:

```bash
remote "az aks get-credentials -n $PRIVATE_AKS -g $RG"
remote "kubectl get node"
```

Clone the repo to have manifest files available:

```bash
remote "git clone https://github.com/yortch/aks-workshop"
```

Run the following commands to create a namespace and deploy hello world application:

```bash
remote "kubectl create namespace helloworld"
remote "kubectl apply -f aks-workshop/manifests/aks-helloworld-basic.yaml -n helloworld"
```

## Validate sample application

Run the following command to verify deployment and service has been created. Re-run command until pod shows a STATUS of Running.

```bash
remote "kubectl get all -n helloworld"
```

Run this command until an `EXTERNAL_IP` is shown for the service (this may take a couple of minutes after creation):

```bash
remote "kubectl get service -n helloworld"
```

Copy the `EXTERNAL_IP` value and run the curl command to confirm the service is reachable on that address:

```bash
remote "curl -L http://<EXTERNAL_IP>"
```

## Cleanup

To cleanup delete resource group:

```bash
az group delete -g $RG
```
