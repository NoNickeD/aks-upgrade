# AKS Upgrade Cluster

This project contains all steps in order to upgrade an existing AKS cluster, with az cli commands.

0. Finding and defining variables
1. Upgrade the control plane with the new version
2. Add new node pools with new version
3. Cordon and drain the old node pool
4. Check the application is up and running
5. Remove the old node pool

## Finding and defining variables

Finding and assigning to:
1. cluster name
2. resource group
3. version of the K8s

```shell
az aks list -o table

export AKS_NAME="test-kube-cluster"
export AKS_RG="test"
export VERSION_OLD="1.21.7"
```

Finding the available version on your region.

```shell
az aks get-versions -l northeurope -o table

export VERSION_NEW="1.22.4"
```

Finding the properly VM size for the K8s cluster.

```shell
az vm list-sizes -l northeurope -o table

# 4vCPU 8GB RAM
export VM_SIZE_SYSTEM="Standard_D2s_v5"
# 16vCPU 64GB RAM
export VM_SIZE_USER="Standard_D8s_v3"
```

## Upgrade the cluster control plane only to the new version

```shell
az aks upgrade --kubernetes-version $VERSION_NEW \
               --control-plane-only \
               --name $AKS_NAME \
               --resource-group $AKS_RG
```


## Add new node pool with new version

Add system node pool with taints to cluster.

```shell
# Creation/Add of system pool to cluster.
az aks nodepool add --name systempool \
                    --cluster-name $AKS_NAME \
                    --resource-group $AKS_RG \
                    --node-count 3 \
                    --node-vm-size $VM_SIZE_SYSTEM \
                    --kubernetes-version $VERSION_NEW \
                    --max-pods 30 \
                    --priority Regular \
                    --zones 1 2 3 \
                    --node-taints CriticalAddonsOnly=true:NoSchedule \
                    --mode System
```

Check that the new system node pool was added / created to the cluster.

```shell
az aks nodepool list --cluster-name $AKS_NAME --resource-group $AKS_RG -o table
```

```shell
az aks nodepool delete --cluster-name $AKS_NAME \
                       --name nodepool1 \
                       --resource-group $AKS_RG \
                       --no-wait
```

Add new user node pool with new K8s version.

```shell
az aks nodepool add \
     --cluster-name $AKS_NAME \
     --resource-group $AKS_RG \
     --name apppool \
     --node-count 3 \
     --node-vm-size $VM_SIZE_USER \
     --kubernetes-version $VERSION_NEW \
     --max-pods 60 \
     --zones 1 2 3 \
     --priority Regular \
     --mode User
```

Check that the new user node pool was added to the cluster.

```shell
az aks nodepool list --cluster-name $AKS_NAME --resource-group $AKS_RG -o table
```

Remove old system nodepool (if exists).

## Cordon and drain the old node pool

Cordon old user node pool.

```shell
kubectl cordon -l agentpool=oldpool
```

Drain old user node pool.

```shell
kubectl drain -l agentpool=oldpool --ignore-daemonsets --delete-emptydir-data
```

## Check the application is up and running

**It should be defined for each namespace separately**

## Remove the old node pool

Remove old user nodepool

```shell
az aks nodepool delete --cluster-name $AKS_NAME \
                       --name oldpool \
                       --resource-group $AKS_RG \
                       --no-wait
```

Check that the old user node pool was deleted from the cluster.

```shell
az aks nodepool list --cluster-name $AKS_NAME --resource-group $AKS_RG -o table
```
