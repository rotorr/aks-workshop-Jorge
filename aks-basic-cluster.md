# Create Basic AKS Cluster

> Estimated Duration: 30 minutes

This exercise will cover deployment of a basic AKS cluster. This will be use as the basis for most subsequent exercises.

**NOTE:** You need to fulfill these [requirements](environment-setup.md) to complete this exercise.

1. Select the region closest to your location. Use **eastus2** for United States workshops, **westeurope** for European workshops. Other regions include: **eastus, westus, canadacentral, westeurope, centralindia, australiaeast**

1. Set your initials.

    ```bash
    INITIALS="abc"
    ```

1. Define global variables

    ```bash
    RESOURCE_GROUP=aks-$INITIALS-rg
    LOCATION=eastus2
    ```

## Create resource group and a basic cluster using Azure CLI

1. Create Resource Group.

    ```bash
    az group create --location $LOCATION \
                    --resource-group $RESOURCE_GROUP
    ```

1. Define variables for AKS cluster.

    ```bash
    AKS_NAME=aks-$INITIALS
    echo "AKS Cluster Name: $AKS_NAME"
    ```

1. Create a simple AKS cluster.

    ```bash
    az aks create --node-count 2 \
                  --name $AKS_NAME  \
                  --resource-group $RESOURCE_GROUP
    ```

    **NOTE:** The creation process will take able 5-10 minutes.

1. Once complete, connect the cluster to your local client machine.

    ```bash
    az aks get-credentials --name $AKS_NAME \
                           --resource-group $RESOURCE_GROUP
    ```

1. Confirm the connection to the cluster.

    ```bash
    kubectl get nodes
    ```

### Create Azure Container Registry (ACR)

ACR will be used in subsequent labs

```bash
ACR_NAME=acr$INITIALS$RANDOM
az acr create -n $ACR_NAME -g $RESOURCE_GROUP --sku Standard
```

Attach ACR to existing AKS cluster

```bash
az aks update -g $RESOURCE_GROUP -n $AKS_NAME --attach-acr $ACR_NAME
```

### Create a new Deployment

The `manifests/aks-helloworld-basic.yaml` file contains a Deployment manifest.

1. Run the following commands to create a namespace and deploy hello world application:

    ```bash
    kubectl create namespace helloworld
    kubectl apply -f manifests/aks-helloworld-basic.yaml -n helloworld
    ```

1. Run the following command to verify deployment and service has been created. Re-run command until pod shows a `STATUS` of **Running** and `EXTERNAL-IP` of the service shows a value.

    ```bash
    kubectl get all -n helloworld
    ```

1. Copy the external IP value or use this command to save it into a variable:

    ```bash
    EXTERNAL_IP=$(kubectl get svc aks-helloworld -n helloworld -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
    ```

1. Run curl command to confirm the service is reachable on that address:

    ```bash
    curl -L http://$EXTERNAL_IP
    ```
