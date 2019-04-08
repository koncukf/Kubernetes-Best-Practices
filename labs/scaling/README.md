# Lab: Scaling Cluster and Applications

In this lab we will scale our application in various ways including scaling our deployment and the AKS cluster.

## Prerequisites

* Complete previous labs:
    * [Azure Kubernetes Service](../create-aks-cluster/README.md)
    * [Build Application Components in Azure Container Registry](../build-application/README.md)
    * [Helm Setup and Deploy Application](../helm-setup-deploy/README.md)

## Instructions

### Scale application manually

1. In this step, we will scale our deployment manually

    ```bash
    kubectl scale deployment service-tracker-ui -n hackfest --replicas=3
    
    ```

    > Note: the `replicas` parameter could be a part of the Helm chart and updated with a `helm upgrade` command. Based on our deployment method, this is more appropriate.

2. Validate the number of pods is now 3

   ```bash
   kubectl get pod -n hackfest
   ```
   
### Horizontal Pod Autoscaler

The Kubernetes Horizontal Pod Autoscaler (HPA) automatically scales the number of pods in a replication controller, deployment or replica set based on observed CPU utilization (or, with custom metrics support, on some other application-provided metrics)

1. Review [hpa.yaml](./hpa.yaml)

2. Deploy the hpa resource

    ```bash
    kubectl apply -f ~/Kubernetes-Best-Practices/labs/scaling/hpa.yaml -n hackfest
    ```
## Hit it with some Load
I've been using Hey

export GOPATH=~/go
export PATH=$GOPATH/bin:$PATH
go get -u github.com/rakyll/hey
hey -z 20m http://<whatever-the-ingress-url-is>

3. Validate the number of pods is now 5 which is our `minReplicas` set with the HPA

4. We could generate traffic to our front end enough to drive CPU utilization up and the HPA would auto-scale up to 10 max

### Cluster Scaling Manual 

1. Scaling is super simple and can be performed in the portal or via the CLI: 
   
    ```bash
    az aks scale --name $CLUSTERNAME --resource-group $RGNAME --node-count 5
    ```

2. Validate that 5 nodes are now available in the cluster


### Cluster Autoscaler 

As resource demands increase, the cluster autoscaler allows your cluster to grow to meet that demand based on constraints you set. The cluster autoscaler (CA) does this by scaling your agent nodes based on pending pods.

> Note: The AKS Cluster Autoscaler is currently in preview. 

1. Follow the steps in this article to configure autoscaling. * [Cluster Autoscaler on Azure Kubernetes Service](https://docs.microsoft.com/en-us/azure/aks/autoscaler)

     Install the aks-preview Azure CLI extension using the az extension add command, as shown in the following example;
     
     ```bash 
     az extension add --name aks-preview
      ```
### Create an AKS cluster and enable the cluster autoscaler     
 
 The following example creates an AKS cluster with virtual machine scale set and the cluster autoscaler enabled, and uses a minimum of 1 and maximum of 3 nodes:     
 
### First create a resource group
      ```bash
      az group create --name myResourceGroup --location canadaeast
      ```
    
### Now create the AKS cluster and enable the cluster autoscaler
      ```bash
      az aks create \
      --resource-group myResourceGroup \
      --name myAKSCluster \
      --kubernetes-version 1.12.4 \
      --node-count 1 \
      --enable-vmss \
      --enable-cluster-autoscaler \
      --min-count 1 \
      --max-count 3   
        ```
### Enable the cluster autoscaler on an existing AKS cluster

You can enable the cluster autoscaler on an existing AKS cluster that meets the requirements as outlined in the preceding Before you begin section. Use the az aks update command and choose to --enable-cluster-autoscaler, then specify a node --min-count and --max-count. The following example enables cluster autoscaler on an existing cluster that uses a minimum of 1 and maximum of 3 nodes:
   
    ```bash
     az aks update \
     --resource-group myResourceGroup \
     --name myAKSCluster \
     --enable-cluster-autoscaler \
     --min-count 1 \
     --max-count 3 
      ``` 
### Disable the cluster autoscale

If you no longer wish to use the cluster autoscaler, you can disable it using the az aks update command. Nodes aren't removed when the cluster autoscaler is disabled.
     
     ```bash
      az aks update \
      --resource-group myResourceGroup \
      --name myAKSCluster \
      --disable-cluster-autoscaler
      
      
## Troubleshooting / Debugging



## Docs / References

* [Horizontal Pod Autoscaler](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale)
