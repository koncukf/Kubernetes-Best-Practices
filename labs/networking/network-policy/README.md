# Lab: Secure traffic between pods using network policies in Azure Kubernetes Service (AKS)

When you run modern, microservices-based applications in Kubernetes, you often want to control which components can communicate with each other. The principle of least privilege should be applied to how traffic can flow between pods in an AKS cluster. For example, you likely want to block traffic directly to backend applications. In Kubernetes, the Network Policy feature lets you define rules for ingress and egress traffic between pods in a cluster.

## Prerequisites

* Complete previous labs:
    * [Azure Kubernetes Service](../../create-aks-cluster/README.md)
    * [Build Application Components in Azure Container Registry](../../build-application/README.md)
    * [Helm Setup and Deploy Application](../../helm-setup-deploy/README.md)

## Overview of network policy

By default, all pods in an AKS cluster can send and receive traffic without limitations. To improve security, you can define rules that control the flow of traffic. For example, backend applications are often only exposed to required frontend services, or database components are only accessible to the application tiers that connect to them.

Network policies are Kubernetes resources that let you control the traffic flow between pods. You can choose to allow or deny traffic based on settings such as assigned labels, namespace, or traffic port. Network policies are defined as a YAML manifests, and can be included as part of a wider manifest that also creates a deployment or service.
To see network policies in action, let's create and then expand on a policy that defines traffic flow as follows:

Deny all traffic to pod.
Allow traffic based on pod labels.
Allow traffic based on namespace.

## Create an AKS cluster and enable network policy
The following example script:
     
     #Creates a virtual network and subnet.
     #Creates an Azure Active Directory (AD) service principal for use with the AKS cluster.
     #Assigns Contributor permissions for the AKS cluster service principal on the virtual network.
     #Creates an AKS cluster in the defined virtual network, and enables network policy.
     
Provide your own secure SP_PASSWORD. If desired, replace the RESOURCE_GROUP_NAME and CLUSTER_NAME variables:
      ```bash
      SP_PASSWORD=mySecurePassword
      RESOURCE_GROUP_NAME=myResourceGroup-NP
      CLUSTER_NAME=myAKSCluster
      LOCATION=canadaeast
      ```

       # Create a resource group
      az group create --name $RESOURCE_GROUP_NAME --location $LOCATION

      # Create a virtual network and subnet
      az network vnet create \
    --resource-group $RESOURCE_GROUP_NAME \
    --name myVnet \
    --address-prefixes 10.0.0.0/8 \
    --subnet-name myAKSSubnet \
    --subnet-prefix 10.240.0.0/16

      # Create a service principal and read in the application ID
      SP_ID=$(az ad sp create-for-rbac --password $SP_PASSWORD --skip-assignment --query [appId] -o tsv)

      # Wait 15 seconds to make sure that service principal has propagated
      echo "Waiting for service principal to propagate..."
   sleep 15

      # Get the virtual network resource ID
      VNET_ID=$(az network vnet show --resource-group $RESOURCE_GROUP_NAME --name myVnet --query id -o tsv)

      # Assign the service principal Contributor permissions to the virtual network resource
      az role assignment create --assignee $SP_ID --scope $VNET_ID --role Contributor

      # Get the virtual network subnet resource ID
      SUBNET_ID=$(az network vnet subnet show --resource-group $RESOURCE_GROUP_NAME --vnet-name myVnet --name myAKSSubnet --query id -o tsv)

      # Create the AKS cluster and specify the virtual network and service principal information
      # Enable network policy using the `--network-policy` parameter
      az aks create \
    --resource-group $RESOURCE_GROUP_NAME \
    --name $CLUSTER_NAME \
    --node-count 1 \
    --kubernetes-version 1.12.4 \
    --generate-ssh-keys \
    --network-plugin azure \
    --service-cidr 10.0.0.0/16 \
    --dns-service-ip 10.0.0.10 \
    --docker-bridge-address 172.17.0.1/16 \
    --vnet-subnet-id $SUBNET_ID \
    --service-principal $SP_ID \
    --client-secret $SP_PASSWORD \
    --network-policy calico

It takes a few minutes to create the cluster. When finished, configure kubectl to connect to your Kubernetes cluster using the az aks get-credentials command. 

      az aks get-credentials --resource-group $RESOURCE_GROUP_NAME --name $CLUSTER_NAME

## Deny all inbound traffic to a pod

   ```bash
   kubectl create namespace development
   kubectl label namespace/development purpose=development
   ```
Now create an example backend pod that runs NGINX. 

    kubectl run backend --image=nginx --labels app=webapp,role=backend --namespace development --expose --port 80 --generator=run-pod/v1
    
To test that you can successfully reach the default NGINX web page, create another pod, and attach a terminal session    
  
    kubectl run --rm -it --image=alpine network-policy --namespace development --generator=run-pod/v1   
    
 Once at shell prompt, use wget to confirm you can access the default NGINX web page:

    ```bash
    wget -qO- http://backend
    ```
Exit out of the attached terminal session. The test pod is automatically deleted:

    exit

## Apply a network policy
  
    ```bash
      kubectl apply -f ./labs/networking/network-policy/backend-policy.yaml    
    ```

Test the network policy
Let's see if you can access the NGINX webpage on the backend pod again. Create another test pod and attach a terminal session:

     kubectl run --rm -it --image=alpine network-policy --namespace development --generator=run-pod/v1
     
Once at shell prompt, use wget to see if you can access the default NGINX web page. This time, set a timeout value to 2 seconds. The network policy now blocks all inbound traffic, so the page cannot be loaded, as shown in the following example:

      $ wget -qO- --timeout=2 http://backend

      wget: download timed out
      
      exit

##    Allow inbound traffic based on a pod label

In the previous section, a backend NGINX pod was scheduled, and a network policy was created to deny all traffic. Now let's create a frontend pod and update the network policy to allow traffic from frontend pods.
Update the network policy to allow traffic from pods with the labels app:webapp,role:frontend and in any namespace.
   
    ```bash
      kubectl apply -f ./labs/networking/network-policy/backend-policy2.yaml    
    ``` 
    
Now schedule a pod that is labeled as app=webapp,role=frontend and attach a terminal session:   
   
    kubectl run --rm -it frontend --image=alpine --labels app=webapp,role=frontend --namespace development --generator=run-pod/v1
   
Once at shell prompt, use wget to see if you can access the default NGINX web page:

     wget -qO- http://backend
     
     
The following example output shows the default NGINX web page returned     
     
      <!DOCTYPE html>
      <html>
      <head>
      <title>Welcome to nginx!</title>
      [...]

Exit out of the attached terminal session. The pod is automatically deleted:

      exit

## Test a pod without a matching label

The network policy allows traffic from pods labeled app: webapp,role: frontend, but should deny all other traffic. Let's test that another pod without those labels can't access the backend NGINX pod. Create another test pod and attach a terminal session:

      ```bash
        kubectl run --rm -it --image=alpine network-policy --namespace development --generator=run-pod/v1
       ```
Once at shell prompt, use wget to see if you can access the default NGINX web page 

      $ wget -qO- --timeout=2 http://backend

      wget: download timed out
      
Exit out of the attached terminal session. The test pod is automatically deleted:
      
      exit  
      
      
 ##   Allow traffic only from within a defined namespace
 
 In the previous examples, you created a network policy that denied all traffic, then updated the policy to allow traffic from pods with a specific label. One other common need is to limit traffic to only within a given namespace. If the previous examples were for traffic in a development namespace, you may want to then create a network policy that prevents traffic from another namespace, such as production, from reaching the pods.
 
    ```bash
    # First, create a new namespace to simulate a production namespace:
    kubectl create namespace production
    kubectl label namespace/production purpose=production 
    # Schedule a test pod in the production namespace that is labeled as app=webapp,role=frontend. Attach a terminal session
    kubectl run --rm -it frontend --image=alpine --labels app=webapp,role=frontend --namespace production --generator=run-pod/v1    
    # Once at shell prompt, use wget to confirm you can access the default NGINX web page:
    wget -qO- http://backend.development    
    ```

    **Sample Output:**
    ```bash
    <!DOCTYPE html>
    <html>
    <head>
    <title>Welcome to nginx!</title>
    [...]
    ```

    ```bash
    # Now exit out of the Pod
    exit
    ```
## Update the network policy

Now let's update the ingress rule namespaceSelector section to only allow traffic from within the development namespace.

      ```bash
       kubectl apply -f ./labs/networking/network-policy/backend-policy-update.yaml    
      ``` 
Now schedule another pod in the production namespace and attach a terminal session:


    ```bash
    # Now schedule another pod in the production namespace and attach a terminal session
    kubectl run --rm -it frontend --image=alpine --labels app=webapp,role=frontend --namespace production --generator=run-pod/v1
   
    # Once at shell prompt, use wget to see the network policy now deny traffic
    $ wget -qO- --timeout=2 http://backend.development

      wget: download timed out
      
    # Exit out of the test pod:
    exit   
    
    # With traffic denied from the production namespace, now schedule a test pod back in the development namespace and attach a terminal session
    kubectl run --rm -it frontend --image=alpine --labels app=webapp,role=frontend --namespace development --generator=run-pod/v1    ```

    # Once at shell prompt, use wget to see the network policy allow the traffic:
    wget -qO- http://backend
    
    
    **Sample Output:**
    ```bash
    <!DOCTYPE html>
    <html>
    <head>
    <title>Welcome to nginx!</title>
    [...]
    ```

    ```bash
    # Now exit out of the Pod
    exit
    ```
## Clean up resources

    kubectl delete namespace production
    kubectl delete namespace development
    

## Troubleshooting / Debugging

* Check to make sure that the namespace that is in the yaml files is the same namespace that the Microservices are deployed to.

## Docs / References

* [Kubernetes Network Policy](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
* [Kube-Router Docs](https://www.kube-router.io/)
* [Kube-Router Repo](https://github.com/cloudnativelabs/kube-router)

#### Next Lab: [Monitoring and Logging](../../monitoring-logging/README.md)
