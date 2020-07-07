# Scale an AKS cluster manually

### Before you begin

- Install Azure CLI `az` and `kubectl` locally or use Azure CloudShell.

### Create an AKS cluster

In order to run an AKS cluster that supports node pools for Windows Server containers, your cluster needs to use a network policy that uses Azure CNI network plugin.
Use the `az aks create` command below to create an AKS cluster named *myAKSCluster*. This command will create the necessary network resources if they don't exist.

```azurecli-interactive
az aks create \
    --resource-group myResourceGroup \
    --name myAKSCluster \
    --node-count 1 \
    --enable-addons monitoring \
    --kubernetes-version 1.16.7 \
    --generate-ssh-keys \
    --network-plugin azure
```

### Scale the cluster nodes

First, get the name of your node pool using the `az aks show` command. 
The following example gets the node pool name for the cluster named myAKSCluster in the myResourceGroup resource group:

```azurecli
az aks show --resource-group myResourceGroup --name myAKSCluster --query agentPoolProfiles
```

Use the az aks scale command to scale the cluster nodes. 
The following example scales a cluster named myAKSCluster to a single node. 
Provide your own --nodepool-name from the previous command, such as nodepool1:

```azurecli
az aks scale --resource-group myResourceGroup --name myAKSCluster --node-count 3 --nodepool-name <your node pool name>
```