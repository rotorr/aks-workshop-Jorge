# Challenge 01 - AKS Private Clusters

## Introduction

This challenge will cover deployment of an AKS cluster fully integrated in a Virtual Network as well as deploying the sample app and configuring ingress.

## Description

You need to fulfill these requirements to complete this challenge:

### Deploy an AKS Cluster

- Deploy an AKS cluster integrated in an existing VNet (you need to create the VNet in advance)
- Deploy as few nodes as possible
- Attach the cluster to existing Azure Container Registry.

**NOTE:** If you do not have "Owner" permissions on your Azure subscription, you will not have permission to attach your AKS cluster to your ACR.

**HINT:** If you decide to use your own ACR with the images for api and web, you must fully qualify the name of your ACR. An image with a non-fully qualified registry name is assumed to be in Docker Hub.

### Deploy the sample application

- Deploy an Azure SQL Database
- Deploy the API and Web containers, expose them over an ingress controller (consider the Application Gateway Ingress Controller, although it is not required).
    - Make sure the links in the section `Direct access to API` of the web page exposed by the Web container are working, as well as the links in the Web menu bar (`Info`, `HTML Healthcheck`, `PHPinfo`, etc)

## Success Criteria

- Verify the application is reachable over the ingress controller, and the API can read the database version successfully
- Verify the links in the `Direct access to API` section of the frontend are working

## Advanced Challenges (Optional)

- Make sure the AKS cluster does not have **any** public IP address
- Configure the Azure SQL Database so that it is only reachable over a private IP address
- Use an open source managed database, such as Azure SQL Database for MySQL or Azure Database for Postgres

## Learning Resources

These docs might help you achieving these objectives:

- [Azure Private Link](https://docs.microsoft.com/azure/private-link/private-link-overview)
- [Restrict AKS egress traffic](https://docs.microsoft.com/azure/aks/limit-egress-traffic)
- [Azure SQL Database](https://docs.microsoft.com/azure/azure-sql/azure-sql-iaas-vs-paas-what-is-overview)
- [AKS Overview](https://docs.microsoft.com/azure/aks/)
- [Application Gateway Ingress Controller](https://docs.microsoft.com/azure/application-gateway/ingress-controller-overview)
- [Create an Nginx ingress controller in AKS](https://docs.microsoft.com/azure/aks/ingress-basic?tabs=azure-cli)
- [Web Application Routing Addon](https://docs.microsoft.com/azure/aks/web-app-routing)

## Solution Guide - Public Clusters and no Firewall Egress

In the `/Solutions/Challenge-02/Public` folder, you will find a set of YAML files that deploy the Whoami sample application to a public AKS cluster.  These YAML files have multiple placeholders in them that need to be replaced with values in order to deploy them to an AKS cluster.

There are a set of Terraform modules located in the [`/Solutions/Challenge-02/Terraform`](./Solutions/Challenge-02/Terraform/) folder that will deploy the Azure resources needed for a public AKS cluster and an Azure SQL Database. If you use the Terraform modules, you will need to update the YAML files with the appropriate values before applying them to the AKS cluster.

Alternatively, the script blocks below in the collapsible section also demonstrate how to complete all steps of this challenge if you choose to deploy a public AKS cluster. The script blocks use the Azure CLI to deploy and configure the Azure resources.

The script blocks also replace the placeholders in the YAML files with the actual values to use before then applying the YAML files to the AKS cluster.

## Solution

Deploying a private AKS cluster with no public IP addresses is a complex task. This is a simplified script for this challenge using a public clusters and no egress traffic filtering through a firewall.

Deploying a private AKS cluster requires multiple Azure resources to be deployed and configured BEFORE deploying AKS, including a VM jumpbox with Azure CLI & kubectl CLI installed to access the private AKS cluster.

The script blocks below in the collapsible section demonstrate how to complete all steps of this challenge to deploy a private AKS cluster. The script blocks use the Azure CLI to deploy and configure the Azure resources.

Here's an example reference article: [Fully Private AKS Clusters Without Any Public IPs: Finally!](https://denniszielke.medium.com/fully-private-aks-clusters-without-any-public-ips-finally-7f5688411184)

This is a script for this challenge using private clusters and filtering egress traffic through firewall:

Export environment variables:

```bash
location=eastus2
rg=rg-aks-workshop-private
aks_name=aks-private
```

## Create Private AKS Cluster

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

## Create Jumpbox VM

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

## Deploy sample application

Clone the repo to have manifest files available:

```bash
remote "git clone https://github.com/yortch/aks-workshop"
```

Run the following commands to create a namespace and deploy hello world application:

```bash
remote "kubectl create namespace helloworld"
remote "cp manifests/aks-helloworld.yaml"
remote "kubectl apply -f aks-workshop/manifests/aks-helloworld.yaml -n helloworld"
remote "kubectl get all -n helloworld"
```
