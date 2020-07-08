# AutoScale an AKS cluster 

### Before you begin

- Install Azure CLI `az` and `kubectl` locally or use Azure CloudShell.

### Create an AKS cluster and enable the cluster autoscaler

If you need to create an AKS cluster, use the `az aks create` command. 
To enable and configure the cluster autoscaler on the node pool for the cluster, use the *--enable-cluster-autoscaler* parameter, and specify a node *--min-count* and *--max-count*.

The following example creates an AKS cluster with a single node pool backed by a virtual machine scale set. It also enables the cluster autoscaler on the node pool for the cluster and sets a minimum of *1* and maximum of *3* nodes:

```azurecli-interactive
# First create a resource group
az group create --name myResourceGroup --location eastus

# Now create the AKS cluster and enable the cluster autoscaler
az aks create \
  --resource-group myResourceGroup \
  --name myAKSCluster \
  --node-count 1 \
  --vm-set-type VirtualMachineScaleSets \
  --load-balancer-sku standard \
  --enable-cluster-autoscaler \
  --min-count 1 \
  --max-count 3
```

### Change the cluster autoscaler settings

To change the node count, use the `az aks update` command.

```azurecli-interactive
az aks update \
  --resource-group myResourceGroup \
  --name myAKSCluster \
  --update-cluster-autoscaler \
  --min-count 1 \
  --max-count 5
```

The above example updates cluster autoscaler on the single node pool in *myAKSCluster* to a minimum of *1* and maximum of *5* nodes.

> [!NOTE]
> The cluster autoscaler makes scaling decisions based on the minimum and maximum counts set on each node pool, but it does not enforce them after updating the min or max counts. 
For example, setting a min count of 5 when the current node count is 3 will not immediately scale the pool up to 5. 
If the minimum count on the node pool has a value higher than the current number of nodes, the new min or max settings will be respected when there are enough unschedulable pods present that would require 2 new additional nodes and trigger an autoscaler event. 
After the scale event, the new count limits are respected.

### Using the autoscaler profile

You can also configure more granular details of the cluster autoscaler by changing the default values in the cluster-wide autoscaler profile. For example, a scale down event happens after nodes are under-utilized after 10 minutes. If you had workloads that ran every 15 minutes, you may want to change the autoscaler profile to scale down under utilized nodes after 15 or 20 minutes. When you enable the cluster autoscaler, a default profile is used unless you specify different settings. The cluster autoscaler profile has the following settings that you can update:

| Setting                          | Description                                                                              | Default value |
|----------------------------------|------------------------------------------------------------------------------------------|---------------|
| scan-interval                    | How often cluster is reevaluated for scale up or down                                    | 10 seconds    |
| scale-down-delay-after-add       | How long after scale up that scale down evaluation resumes                               | 10 minutes    |
| scale-down-delay-after-delete    | How long after node deletion that scale down evaluation resumes                          | scan-interval |
| scale-down-delay-after-failure   | How long after scale down failure that scale down evaluation resumes                     | 3 minutes     |
| scale-down-unneeded-time         | How long a node should be unneeded before it is eligible for scale down                  | 10 minutes    |
| scale-down-unready-time          | How long an unready node should be unneeded before it is eligible for scale down         | 20 minutes    |
| scale-down-utilization-threshold | Node utilization level, defined as sum of requested resources divided by capacity, below which a node can be considered for scale down | 0.5 |
| max-graceful-termination-sec     | Maximum number of seconds the cluster autoscaler waits for pod termination when trying to scale down a node. | 600 seconds   |
| balance-similar-node-groups | Detect similar node pools and balance the number of nodes between them | false |

> [!IMPORTANT]
> The cluster autoscaler profile affects all node pools that use the cluster autoscaler. You can't set an autoscaler profile per node pool.

### Install aks-preview CLI extension

To set the cluster autoscaler settings profile, you need the *aks-preview* CLI extension version 0.4.30 or higher. Install the *aks-preview* Azure CLI extension using the [az extension add][az-extension-add] command, then check for any available updates using the [az extension update][az-extension-update] command:

```azurecli-interactive
# Install the aks-preview extension
az extension add --name aks-preview

# Update the extension to make sure you have the latest version installed
az extension update --name aks-preview
```

### Set the cluster autoscaler profile on an existing AKS cluster

Use the `az aks update` command with the *cluster-autoscaler-profile* parameter to set the cluster autoscaler profile on your cluster. The following example configures the scan interval setting as 30s in the profile.

```azurecli-interactive
az aks update \
  --resource-group myResourceGroup \
  --name myAKSCluster \
  --cluster-autoscaler-profile scan-interval=30s
```

When you enable the cluster autoscaler on node pools in the cluster, those clusters will also use the cluster autoscaler profile. For example:

```azurecli-interactive
az aks nodepool update \
  --resource-group myResourceGroup \
  --cluster-name myAKSCluster \
  --name mynodepool \
  --enable-cluster-autoscaler \
  --min-count 1 \
  --max-count 3
```

> [!IMPORTANT]
> When you set the cluster autoscaler profile, any existing node pools with the cluster autoscaler enabled will start using the profile immediately.

### Set the cluster autoscaler profile when creating an AKS cluster

You can also use the *cluster-autoscaler-profile* parameter when you create your cluster. For example:

```azurecli-interactive
az aks create \
  --resource-group myResourceGroup \
  --name myAKSCluster \
  --node-count 1 \
  --enable-cluster-autoscaler \
  --min-count 1 \
  --max-count 3 \
  --cluster-autoscaler-profile scan-interval=30s
```

The above command creates an AKS cluster and defines the scan interval as 30 seconds for the cluster-wide autoscaler profile. The command also enables the cluster autoscaler on the initial node pool, sets the minimum node count to 1 and the maximum node count to 3.