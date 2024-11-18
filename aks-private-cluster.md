# Create AKS Private Clusters

> Estimated Duration: 60 minutes

## Introduction

This challenge will cover deployment of a private AKS cluster fully integrated in a Virtual Network as well as deploying the sample app and configuring ingress.

You need to fulfill these [requirements](environment-setup.md) to complete this challenge

## Success Criteria

- Verify the application is reachable over the ingress controller

## Learning Resources

These docs will help you achieving these objectives:

- [AKS Overview](https://docs.microsoft.com/azure/aks/)
- [Web Application Routing Addon](https://docs.microsoft.com/azure/aks/web-app-routing)

## Solution

Deploying a private AKS cluster is a complex task. It requires multiple Azure resources to be deployed and configured BEFORE deploying AKS, including a VM jumpbox with Azure CLI & kubectl CLI installed to access the private AKS cluster.

The script blocks below in the sections below demonstrate how to complete all steps of this challenge to deploy a private AKS cluster. The script blocks use the Azure CLI to deploy and configure the Azure resources.

### Create resource group and virtual network

Export environment variables required for resource group:

```bash
location=eastus2
rg=rg-aks-workshop-private
aks_name=aks-private
```

Login to Azure and create resource group:

```bash
az login
az group create --name $rg --location $location
```

Export variables for the virtual network for the AKS cluster:

```bash
vnet_name=aks-vnet
vnet_prefix=10.13.0.0/16
aks_subnet_name=aks-subnet
aks_subnet_prefix=10.13.76.0/24
```

Create the virtual network for the AKS cluster:

```bash
az network vnet create -g $rg -n $vnet_name --address-prefix $vnet_prefix -l $location
az network vnet subnet create -g $rg -n $aks_subnet_name --vnet-name $vnet_name --address-prefix $aks_subnet_prefix
aks_subnet_id=$(az network vnet subnet show -n $aks_subnet_name --vnet-name $vnet_name -g $rg --query id -o tsv)
```

### Create Private AKS Cluster

NOTE: if using MinGW client such as git-bash in Windows, export this variable to avoid converting variables with a leading `/` into system paths:

```bash
export MSYS_NO_PATHCONV=1
```

Create Private AKS cluster with Azure CNI Overlay, app routing and managed identity enabled:

```bash
az aks create -g $rg -n $aks_name -l $location \
    --generate-ssh-keys --enable-private-cluster \
    --vnet-subnet-id $aks_subnet_id \
    --network-plugin azure --network-policy cilium \
    --network-plugin-mode overlay --network-dataplane cilium \
    --enable-managed-identity --enable-app-routing
```

Cluster creation will take a few minutes. Once created, attempt to connect and you will notice it will fail due to being a private cluster:

```bash
# Cluster-info
az aks get-credentials -n $aks_name -g $rg --overwrite
kubectl get node
```

To validate connectivity you can navigate to the created aks cluster resource in Azure Portal and clicking on [**Run command**](https://learn.microsoft.com/en-us/azure/aks/access-private-cluster?source=recommendations&tabs=azure-cli#run-commands-on-your-aks-cluster) under **Kubernetes Resources**

### Create Jumpbox VM

Next create Jumpbox VM to connect to the Private AKS cluster. You will create a VM in the same vnet and install kubectl to have access to the API.

Export environment variables:

```bash
# Variables
vm_name=vm-jumpbox
vm_nsg_name="${vm_name}-nsg"
vm_sku=Standard_B2ms
vm_subnet_name=vm-subnet
vm_subnet_prefix=10.13.2.0/24
image_urn=$(az vm image list -f "ubuntu-24_04-lts" -s "server" -l "$location" --query '[0].urn' -o tsv)
```

Create Subnet for the jumpbox:

```bash
az network vnet subnet create -n $vm_subnet_name --vnet-name $vnet_name -g "$rg" --address-prefixes $vm_subnet_prefix
```

Create jumpbox VM:

```bash
az vm create -n $vm_name -g $rg -l $location --image $image_urn --size $vm_sku --generate-ssh-keys \
  --vnet-name $vnet_name --subnet $vm_subnet_name \
  --assign-identity --admin-username azureuser \
  --nsg $vm_nsg_name --nsg-rule SSH
```

Enable the Microsoft Entra login VM extension:

```bash
az vm extension set --publisher Microsoft.Azure.ActiveDirectory \
    --name AADSSHLoginForLinux -g $rg --vm-name $vm_name
```

Grant VM admin login to your Entra login:

```bash
username=$(az account show --query user.name --output tsv)
rg_id=$(az group show --resource-group $rg --query id -o tsv)
```

Next assign role:

```bash
az role assignment create --role "Virtual Machine Administrator Login" --assignee $username --scope $rg_id
```

NOTE: if your Entra domain and login username do not match then use these command instead:

```bash
userid=$(az ad user list --filter "mail eq '$username'" --query [0].id -o tsv)
az role assignment create --role "Virtual Machine Administrator Login" --assignee-object-id $userid --scope $rg_id
```

Create managed identity and assign role to be able to login to AKS from jumpbox (only needed if your subscription does not allow you to log in via cli using device code):

```bash
# Managed identity
vm_identity_name=${vm_name}-identity
az identity create -g $rg -n $vm_identity_name
az vm identity assign -n $vm_name -g $rg --identities ${vm_name}-identity
vm_identity_clientid=$(az identity show -n ${vm_name}-identity -g $rg --query clientId -o tsv)
vm_identity_principalid=$(az identity show -n ${vm_name}-identity -g $rg --query principalId -o tsv)
vm_identity_id=$(az identity show -n ${vm_name}-identity -g $rg --query id -o tsv)
rg_id=$(az group show -n $rg --query id -o tsv)
az role assignment create --assignee $vm_identity_principalid --role Contributor --scope $rg_id
aks_id=$(az aks show --resource-group $rg --name $aks_name --query id --output tsv)
az role assignment create --assignee $vm_identity_principalid --role "Azure Kubernetes Service RBAC Cluster Admin" --scope $aks_id
```

Output value of `vm_identity_id` and copy it so that it can be pasted when connected to jumpbox

```bash
echo $vm_identity_id
```

Create remote alias to SSH into jumpbox using:

```bash
alias remote="az ssh vm -n $vm_name -g $rg"
```

Run the following commands to install kubectl and az login and confirm access:

```bash
remote "curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash"
remote "sudo az aks install-cli"
remote "az login --identity -u $vm_identity_id"
remote "az aks get-credentials -n $aks_name -g $rg"
remote "kubectl get node"
```

### Deploy sample application

Clone the repo to have manifest files available:

```bash
remote "git clone https://github.com/yortch/aks-workshop"
```

Run the following commands to create a namespace and deploy hello world application:

```bash
remote "kubectl create namespace helloworld"
remote "cp manifests/aks-helloworld.yaml"
remote "kubectl apply -f aks-workshop/manifests/aks-helloworld.yaml -n helloworld"
```

Run the following command to verify deployment and service has been created. Re-run command until pod shows a STATUS of Running.

```bash
remote "kubectl get all -n helloworld"
```

Run this command until an ADDRESS is shown for the ingress (this may take a couple of minutes after creation):

```bash
remote "kubectl get ingress -n helloworld"
```

Run curl command to confirm the service is reachable on that address:

```bash
remote "curl -L http://<ADDRESS_IP>"
```
