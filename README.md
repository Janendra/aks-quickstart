# AKS Walkthrough

## Scenario
In this walkthrough, we're going to create an AKS cluster and deploy services to the cluster. Much of the code is based on <https://docs.microsoft.com/en-us/azure/aks/kubernetes-walkthrough> but additional details and scenarios are covered.

After creating the cluster, we will deploy a backend service based on the standard Redis Docker container, demonstrating how to connect to containers in the cluster via port forwarding. All deployments are done using the "declaritive" approach - read more here: <https://kubernetes.io/docs/concepts/overview/object-management-kubectl/declarative-config/>

Next we will deploy the web front end which uses the Redis backend to count "votes". This will automatically create an Azure Public IP address as well as an Azure Load Balancer.

Next we will deploy a simple Go Web app as a DaemonSet. This will create another Azure Public IP address and reconfigure the load balancer to support both IPs.

Next we will scale the cluster from one node to three nodes. This will cause the Go Web App to create 3 instances as one per node is the default for DaemonSets. The other services will not change. This step is optional.

Next we will install Helm onto the cluster and use a Helm Chart to deploy a Redis cluster. We will modify the web app to use the new cluster and delete the original backend deployment.

Lastly, we'll delete everything.

The walk through takes about 45 minutes.

## Let's get started

AKS is not currently available in all regions, so we'll need to pick the correct region. The default in the script is Central US. We also need to verify that whatever size VMs we use for nodes are available in that region as well.

### List of regions where AKS is available

<https://docs.microsoft.com/en-us/azure/aks/container-service-quotas>

### List of VM Sizes in that region

```
# change centralus to a different region if desired
az vm list-sizes -l centralus -o table | grep s_v3
```

### Set environment variables

```
# Change these if desired
AKSSUB=k8s
AKSLOC=centralus
AKSSIZE=Standard_D2s_v3

AKSRG=aks
AKSNAME=aks

# setup an alias (optional)
alias k=kubectl
```

### Login and select your Azure subscription

```
az login
az subscrition set -s $AKSSUB
```

### Create a resource group and AKS Cluster

```
az group create -l $AKSLOC -g $AKSRG

# this takes a while
az aks create -g $AKSRG -n $AKSNAME -c 1 -s $AKSSIZE
```

### What this did
If you check the Azure portal, you will see that this command created two resource groups - AKS and MC_AKSRG_AKSNAME_AKSLOC. The AKS resource group contains the AKS service (Kubernetes Controller). The other resource group contains the k8s nodes. Here is a screen shot of what is created.

![Initial node resource group](images/aks-node-initial.PNG)

### Get the k8s credentials and save in ~/.kube/config

```
az aks get-credentials -g $AKSRG -n $AKSNAME

# Note: if you run this quickstart repeatedly, you will need to edit / remove ~/.kube/control before running this command
```

### Wait for the node(s) to be ready

```
watch kubectl get nodes
```

### Create and deploy backend (Redis)

```
kubectl apply -f backend

# Wait for the app to start
watch kubectl get deploy,pods
```

## Connect to the Redis container

```
# Start port forarding in background
kubectl port-forward svc/backend 6379:6379 &

# wait for port to be forwarded

# Run some Redis commands
bin/redis-cli

set Dogs 10
set Cats 1

get Dogs
get Cats

exit

# Stop Port Forwarding
fg
<ctl> c
```

### Create and deploy frontend web app

```
kubectl apply -f frontend

# Get the public IP
watch kubectl get svc frontend

# browse to public IP to test app
```

### Node Resource Group

Notice that AKS added a Public IP and a Load Balancer after you deployed the frontend service. The YAML in frontend/svc.yaml specifies "LoadBalancer" as the type of service.

![screenshot](images/aks-node-frontend.PNG)

### Deploy a simple go web app

```
kubectl apply -f fe2

# Wait for service to start
watch kubectl get svc,pods

# Browse to public IP
```

### Node Resource Group

Notice that AKS added a Public IP and reconfigured the Load Balancer after you deployed the frontend service. The YAML in fe2/svc.yaml specifies "LoadBalancer" as the type of service.

![screenshot](images/aks-node-fe2.PNG)

### Scale the k8s cluster to 3 nodes

```
# this step is optional
# the Helm deployment will work without doing this step
az aks scale --no-wait -g $AKSRG -n $AKSNAME -c 3

# Wait for nodes to deploy
watch kubectl get nodes,pods
```

### Install Helm cli

```
Azure Cloud Shell: already installed
Ubuntu: sudo snap install helm --classic
Mac: brew install kubernetes-helm
```

### Intialize Helm on cluster

```
# Add RBAC role for Tiller
kubectl apply -f helm/rbac-helm.yaml

# Initialize Helm
helm init --service-account tiller --upgrade

# Wait for tiller to be available
watch kubectl -n kube-system get deploy
```

### Install redis from Helm

```
helm install --values helm/redis.yaml --name backend stable/redis 

# Wait for pod to be ready
watch kubectl get pods

# Start port forarding in background
kubectl port-forward svc/backend-redis-master 6379:6379 &

# Run some Redis commands
bin/redis-cli

set Dogs 100
set Cats 2

# Stop Port Forwarding
fg

# Press <ctl> c
```

### Change frontend to use Helm deployed service

```
nano frontend/deploy.yaml

# Change "backend" to "backend-redis-master"
# Save

kubectl apply -f frontend/deploy.yaml

# revert the file
git checkout frontend/deploy.yaml

# Wait for deploy / pods
watch kubectl get deploy,pods
```

### Delete original backend service

```
kubectl delete -f backend

# Wait for service to be deleted
watch kubectl get svc,pods
```

### Node Resource Group

Notice that AKS added a disk image to the node Resource Group. The following YAML sets up Redis Persistance in helm/redis.yaml

```
persistence:
    enabled: true
    path: /data
    subPath: ""
    accessModes:
    - ReadWriteOnce
    size: 8Gi
```

![screenshot](images/aks-node-fe2.PNG)

### Clean up

```
# Delete resource groups
az group delete -y --no-wait -g $AKSRG
az group delete -y --no-wait -g MC_${AKSRG}_${AKSNAME}_${AKSLOC}
az group list -o table

# Only remove this file if this is the only k8s cluster in the file. Otherwise, edit the file and remove the key information

rm ~/.kube/config
```
