# Autoscaling using Horizontal Pod Autoscaler and Cluster Autoscaler

> Estimated Duration: 60 minutes
> **NOTE:** You need to fulfill these [requirements](environment-setup.md) and [AKS Basic Cluster](aks-basic-cluster.md) to complete this exercise.

For autoscaling you are given the following requirements:

* Cluster autoscaler should be enabled on both the System and User mode nodepools
  * User pool should have a min count of 1 and a max count of 3
* Since we are rapid testing we want the cluster autoscaler to:
  * Scan every 5 seconds for scale up
  * Scale down when a node is not needed for 1 minute
  * Allow scale down 1m after scale up

You are asked to complete the following tasks:

1. Configure autoscaling for user nodepool based on the requirements above
2. Validate pod autoscaling is working
3. Validate cluster autoscaling is working

## Configure Autoscaling on Nodepools

In prior steps we created a [basic AKS cluster](aks-basic-cluster.md). You need to enable the cluster autoscaler. One of the requirements is to adjust the autoscaler settings, which we can do using the AKS [Cluster Autoscale Profile](https://learn.microsoft.com/en-us/azure/aks/cluster-autoscaler?tabs=azure-cli#use-the-cluster-autoscaler-profile).

Set the Environment Variables used in the lookups

```bash
INITIALS=abc
RG=aks-$INITIALS-rg
CLUSTER_NAME=aks-$INITIALS
```

First, get the nodepool name:

```bash
NODEPOOL_NAME=$(az aks nodepool list -g $RG --cluster-name $CLUSTER_NAME --query '[].name' -o tsv)
```

Based on the requirements, enable the cluster autoscaler on the node pool to min of 1 and max of 3

```bash
az aks nodepool update --resource-group $RG --cluster-name $CLUSTER_NAME \
--name $NODEPOOL_NAME --enable-cluster-autoscaler --min-count 1 --max-count 3
```

You can confirm the config was applied with the following command

```bash
az aks nodepool show -g $RG --cluster-name $CLUSTER_NAME -n $NODEPOOL_NAME -o yaml | grep 'enableAutoScaling\|minCount\|maxCount'

enableAutoScaling: true
maxCount: 3
minCount: 1
```

## Update Cluster Autoscale Profile

Update the cluster with the new autoscale profile settings

```bash
az aks update -g $RG -n $CLUSTER_NAME \
--cluster-autoscaler-profile "scan-interval=30s,scale-down-unneeded-time=1m,scale-down-delay-after-add=1m"
```

You can verify your updates with the following command:

```bash
az aks show -g $RG -n $CLUSTER_NAME -o yaml --query autoScalerProfile
```

## Deploy Horizonal Pod Autoscaler (HPA)

To test the Cluster Autoscaler, we first need to add an HPA we need to create a deployment and a service with resource limits set, and then push the app pods beyond those limits. You can use any image for this, as long as you understand the resource usage characteristics so that you can appropriately set the HPA configuration.

1. Confirm the `helloword` deployment is running

    ```bash
    kubectl get all -n helloworld
    ```

1. In case it is not running, run the following commands to create a namespace and deploy hello world application:

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

1. Deploy the Horizontal Pod Autoscaler

    ```bash
    kubectl apply -f manifests/hpa.yaml -n helloworld
    ```

1. In the current terminal watch the status of the HPA and the pod count:

    ```bash
    kubectl get hpa,pods,nodes -n helloworld
    ```

## Test the Cluster Autoscaler

The Cluster Autoscaler is looking for pods that are in a "Pending" state because there aren't enough nodes to handle the requested resources. To test it, we can just play around with the cpu request size and the max replicas in the HPA. Lets set the request and limit size to 800m cores. This should cause the HPA to create up to 5 pods under load, so we'll quickly spill over the current single node.

Run the command below to set the request and limits for the deployment to 800 millicores (0.8 cores):

```bash
kubectl patch deployment aks-helloworld -n helloworld \
-p '{"spec": {"template": {"spec": {"containers": [{"name": "aks-helloworld", "resources": {"requests": {"cpu": "800m"}, "limits": {"cpu": "800m"}}}]}}}}'
```

Check the number of nodes:

```bash
kubectl get nodes
```

If there are already 3 nodes running, one way to scale down temporarily is to run this command (note that it may take a few minutes for the nodes count to reduce):

```bash
az aks nodepool update --resource-group $RG --cluster-name $CLUSTER_NAME \
--name $NODEPOOL_NAME --update-cluster-autoscaler --min-count 1 --max-count 1
```

Once you confirm there is only one node ready, then run this command to revert the autoscaler settings to a max of 3 nodes:

```bash
az aks nodepool update --resource-group $RG --cluster-name $CLUSTER_NAME \
--name $NODEPOOL_NAME --update-cluster-autoscaler --min-count 1 --max-count 3
```

Confirm the settings have been reverted:

```bash
az aks nodepool show -g $RG --cluster-name $CLUSTER_NAME -n $NODEPOOL_NAME -o yaml | grep 'enableAutoScaling\|minCount\|maxCount'
```

On a separate terminal, run and shell into an Ubuntu pod:

```bash
kubectl run -it --rm ubuntu --image=ubuntu -- bash
```

Run the following in the pod to install ApacheBench:

```bash
apt update && apt install -y apache2-utils 
```

Run a load against the external IP of your "helloworld" service (keep the trailing `/` in the url below):

```bash
ab -t 240 -c 50 -n 200000 http://<EXTERNAL_IP>/
```

You should get output like the following:

```bash
Benchmarking 10.140.0.5 (be patient)
Completed 10000 requests
Completed 20000 requests
Completed 30000 requests
Completed 40000 requests
Completed 50000 requests
Completed 60000 requests
Completed 70000 requests
Completed 80000 requests
Completed 90000 requests
Completed 100000 requests
Finished 100000 requests
```

Next, list the HPA, pods, and nodes:

```bash
kubectl get hpa,pods,nodes -n helloworld
```

You should see the targets in the HPA output increasing. Once it reaches over 50%, then the replicas should increase.

```bash
NAME                                                 REFERENCE                   TARGETS           MINPODS   MAXPODS   REPLICAS   AGE
horizontalpodautoscaler.autoscaling/aks-helloworld   Deployment/aks-helloworld   0%/50%, 29%/50%   1         5         4          44h

NAME                                  READY   STATUS    RESTARTS   AGE
pod/aks-helloworld-67f4dc9d98-wnvwr   1/1     Running   0          14m
pod/aks-helloworld-67f4dc9d98-zr2rs   1/1     Running   0          13m
pod/aks-helloworld-bd8fc7bc6-gr8j4    0/1     Pending   0          6m21s
pod/aks-helloworld-bd8fc7bc6-zhvbk    1/1     Running   0          6m21s
```

If you see pods in Pending state that should eventually trigger the creation of new nodes. Check the status of the pod using:

```bash
kubectl get pod <pod-name> -o yaml -n helloworld
```

You should see a status with the error message below:

```yaml
status:
  conditions:
  - lastProbeTime: null
    lastTransitionTime: "2024-11-22T18:20:44Z"
    message: '0/3 nodes are available: 1 node(s) had untolerated taint {ToBeDeletedByClusterAutoscaler:
      1732299539}, 2 Insufficient cpu. preemption: 0/3 nodes are available: 1 Preemption
      is not helpful for scheduling, 2 No preemption victims found for incoming pod.'
    reason: Unschedulable
    status: "False"
    type: PodScheduled
  phase: Pending
```

Alternatively you may check the events in the namespace using:

```bash
kubectl events -n helloworld
```

And you should see a similar warning:

```txt
Warning   FailedScheduling               Pod/aks-helloworld-bd8fc7bc6-lvkrc       0/3 nodes are available: 1 node(s) had untolerated taint {ToBeDeletedByClusterAutoscaler: 1732299539}, 2 Insufficient cpu. preemption: 0/3 nodes are available: 1 Preemption is not helpful for scheduling, 2 No preemption victims found for incoming pod.
```

The pending pods should increase the provisioned nodes up to the maximum (3). If there are no new nodes allocated, check the node cpu requests and limits using:

```bash
kubectl describe nodes | grep -E -A 6 "Resource|Requests|Limits"
```

Cpu limits may be over 100%, but the requests should be under 100% for all nodes:

```txt
  Resource           Requests     Limits
  --------           --------     ------
  cpu                1220m (64%)  1720m (90%)
  memory             540Mi (10%)  2786Mi (55%)
```

Depending on the node size, if the requests % is not high enough then you may need to increase the request/limit size of the deployment, e.g. to 1000m cores.

```bash
kubectl patch deployment aks-helloworld -n helloworld \
-p '{"spec": {"template": {"spec": {"containers": [{"name": "aks-helloworld", "resources": {"requests": {"cpu": "1000m"}, "limits": {"cpu": "1000m"}}}]}}}}'
```

Rerun the load test if needed in your apachebench terminal:

```bash
ab -t 240 -c 50 -n 200000 http://EXTERNAL_IP/
```

## Tuning

As you probably saw above, CPU based scaling can sometimes be a bit slow. There are a lot of factors that come into play that you can tune. As you saw above, there's the autoscaler profile settings that can be adjusted. To improve the time taken to add a node to the cluster, you can look at the [autoscale mode](https://docs.microsoft.com/en-us/azure/aks/scale-down-mode), which will let you scale down by stopping but not deleting nodes (aka deallocation mode). This way a new node will not need to be provisioned, but rather an existing deallocated node just needs to be started.

## Cleanup

Before moving on to next lab, disable cluster autoscaler:

```bash
az aks nodepool update --resource-group $RG --cluster-name $CLUSTER_NAME --name $NODEPOOL_NAME --disable-cluster-autoscaler
```
