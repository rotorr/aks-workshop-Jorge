# AKS Cluster Upgrade

> Estimated Duration: 60 minutes
> Requires AKS cluster created in: [basic AKS cluster](aks-basic-cluster.md).

You are given the following requirements:

* Initial testing should use a manual upgrade process
* The application pod count should never go below 1 during an upgrade
* Day 1 (simulated): Due to a critical OS level CVE you've been asked to upgrade the node pool **NODE IMAGE ONLY**
* Day 2 (simulated): Due to a critical Kubernetes level CVE you've been asked to upgrade the control plane and the node pool Kubernetes version to the next incremental version (major or minor)
* Day 3 (simulated): To take advantage of some new Kubernetes features you've been asked to upgrade the user pool Kubernetes version to the next incremental version (major or minor)
  
You were asked to complete the following tasks:

1. Increase the deployment replica count to 2
2. Deploy the necessary config to ensure the application pod count never dips below 1 pod
3. Check the available upgrade versions for Kubernetes and Node Image
4. Upgrade the system pool node image
5. Upgrade the AKS control plane and node pool Kubernetes version
6. Upgrade the node pool Kubernetes version
7. **Bonus Tasks:** Enable Automatic Upgrades to the 'patch' channel and set a Planned Maintenance Window (preview) for Saturdays at 1am

## Setup

In a terminal, export variables required for this lab (if not already exported):

```bash
INITIALS=abc
CLUSTER_NAME=aks-$INITIALS
RG=aks-$INITIALS-rg
```

If not already connected, connect to the cluster from your local client machine.

```bash
az aks get-credentials --name $CLUSTER_NAME -g $RG
```

### Deploy helloworld application

If not already deployed, then proceed to deploy the aks-helloworld application.

1. Run the following commands to create a namespace and deploy hello world application:

    ```bash
    kubectl create namespace helloworld
    kubectl apply -f manifests/aks-helloworld-basic.yaml -n helloworld
    ```

1. Run the following command to verify deployment and service has been created.

    ```bash
    kubectl get all -n helloworld
    ```

## Deploy pod disruption budget

Kubernetes provides [Pod Disruption Budgets](https://kubernetes.io/docs/tasks/run-application/configure-pdb/) as a mechanism to ensure a minimum pod count during [Disruptions](https://kubernetes.io/docs/concepts/workloads/pods/disruptions/).

We need to ensure that there is minimum of one pod, by deploying the following pdb:
Deply the pdb.

```bash
kubectl apply -f manifests/pdb.yaml -n helloworld
```

Confirm the pdb has been deployed:

```bash
kubectl get pdb
```

## Upgrade the node pool image

Get the node pool name:

```bash
NODEPOOL_NAME=$(az aks nodepool list -g $RG --cluster-name $CLUSTER_NAME --query '[].name' -o tsv)
```

Get the node pool image version

```bash
az aks nodepool show -g $RG --cluster-name $CLUSTER_NAME --nodepool-name $NODEPOOL_NAME -o tsv --query nodeImageVersion
AKSUbuntu-2204gen2containerd-202410.27.0
```

Get the latest node pool node image version

```bash
az aks nodepool get-upgrades -g $RG --cluster-name $CLUSTER_NAME --nodepool-name $NODEPOOL_NAME -o tsv --query latestNodeImageVersion
AKSUbuntu-2204gen2containerd-202411.03.0
```

In the above check you may have found that you're already running the latest node image version. If so, no action is needed. If not, you can upgrade as follows (this will take 5-10 minutes):

```bash
az aks nodepool upgrade --resource-group $RG --cluster-name $CLUSTER_NAME \
--name $NODEPOOL_NAME --node-image-only 
```

On a separate terminal, you can check the nodes as they get upgraded (this will take a few minutes):

```bash
kubectl get nodes -o wide
```

Additionally it is recommended you check the events for errors.

```bash
kubectl events
```

You should see nodes getting drained and then upgraded

```txt
Normal    Drain                     Node/aks-nodepool1-33528281-vmss000009   Draining node: aks-nodepool1-33528281-vmss000009
Normal    Upgrade                   Node/aks-nodepool1-33528281-vmss000008   Successfully upgraded node: aks-nodepool1-33528281-vmss000008
Normal    Upgrade                   Node/aks-nodepool1-33528281-vmss000008   Successfully reimaged node: aks-nodepool1-33528281-vmss000008
Normal    NodeNotSchedulable        Node/aks-nodepool1-33528281-vmss000009   Node aks-nodepool1-33528281-vmss000009 status is now: NodeNotSchedulable
Normal    Upgrade                   Node/aks-nodepool1-33528281-vmss000009   Deleting node aks-nodepool1-33528281-vmss000009 from API server
Normal    RemovingNode              Node/aks-nodepool1-33528281-vmss000009   Node aks-nodepool1-33528281-vmss000009 event: Removing Node aks-nodepool1-33528281-vmss000009 from Controller
```

Once the nodes have been reimaged check the image version to confirmed they have been upgraded to the latest image version:

```bash
az aks nodepool show -g $RG --cluster-name $CLUSTER_NAME --nodepool-name $NODEPOOL_NAME -o tsv --query nodeImageVersion
```

### Upgrade the node pool Kubernetes version

Check versions available:

```bash
az aks get-upgrades -g $RG -n $CLUSTER_NAME
```

During this lab creation, the versions available are:

```json
    "kubernetesVersion": "1.29.9",
    "name": null,
    "osType": "Linux",
    "upgrades": [
      {
        "isPreview": null,
        "kubernetesVersion": "1.30.5"
      },
      {
        "isPreview": null,
        "kubernetesVersion": "1.30.4"
      },
      {
        "isPreview": null,
        "kubernetesVersion": "1.30.3"
      },
      {
        "isPreview": null,
        "kubernetesVersion": "1.30.2"
      },
      {
        "isPreview": null,
        "kubernetesVersion": "1.30.1"
      },
      {
        "isPreview": null,
        "kubernetesVersion": "1.30.0"
      }
```

To upgrade the kubernetes version, the command is the same as above without the --node-image-only tag. Upgrade the control plane to the next minor version available, e.g. from 1.29.9 to 1.30.0:

```bash
az aks upgrade -g $RG -n $CLUSTER_NAME \
--control-plane-only --kubernetes-version 1.30.0
```

Once completed check the control plane version (server version is the control plane version):

```bash
kubectl version
Client Version: v1.29.1
Server Version: v1.30.0
```

Start the node pool kubernetes version upgrade:

```bash
az aks nodepool upgrade --resource-group $RG --cluster-name $CLUSTER_NAME --name $NODEPOOL_NAME --kubernetes-version 1.30.0
```

Use the same commands to monitor progress on a separate window:

```bash
kubectl get nodes -o wide
kubectl events
```

You should see one node upgraded at a time:

```txt
NAME                                STATUS     ROLES    AGE   VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION      CONTAINER-RUNTIME
aks-nodepool1-33528281-vmss000008   Ready      <none>   21m   v1.29.9   10.224.0.6    <none>        Ubuntu 22.04.5 LTS   5.15.0-1074-azure   containerd://1.7.23-1
aks-nodepool1-33528281-vmss00000b   Ready      <none>   86s   v1.30.0   10.224.0.5    <none>        Ubuntu 22.04.5 LTS   5.15.0-1074-azure   containerd://1.7.23-1
aks-nodepool1-33528281-vmss00000c   NotReady   <none>   94s   v1.30.0   10.224.0.8    <none>        Ubuntu 22.04.5 LTS   5.15.0-1074-azure   containerd://1.7.23-1
```

In the above, notice all the single instance terminations and think about the impact that could have on your running application
It would eventually make sense to increase all of their replica counts and implement PodDisruptionBudgets for those as well
but thats out of scope for this excercise.

## BONUS: Enable Automatic Upgrades to the 'patch' channel and set a Planned Maintenance Window

While the above process is actually pretty easy and could be automated through a ticketing system without much effort, for lower environments (dev/test) you may consider letting AKS manage the version upgrades for you. You can also specify the upgrade window.

Check the current auto upgrade profile (output should be None)

```bash
az aks show -g $RG -n $CLUSTER_NAME -o tsv --query autoUpgradeProfile
NodeImage       None
```

Enable auto upgrades to the 'patch' channel

```bash
az aks update --resource-group $RG --name $CLUSTER_NAME --auto-upgrade-channel patch
```

Check the auto upgrade profile again

```bash
az aks show -g $RG -n $CLUSTER_NAME -o tsv --query autoUpgradeProfile
patch
```

Create the planned maintenance configuration for Saturdays at 1am

```bash
az aks maintenanceconfiguration add -g $RG --cluster-name $CLUSTER_NAME \
--name default --weekday Saturday  --start-hour 1
```

Show the current maintenance window configuration

```bash
az aks maintenanceconfiguration show -g $RG --cluster-name $CLUSTER_NAME \
--name default -o yaml

name: default
notAllowedTime: null
systemData: null
timeInWeek:
- day: Saturday
  hourSlots:
  - 1
type: null
```
