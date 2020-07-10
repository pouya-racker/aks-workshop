# Create and use a persistent volume with Azure disks

### Before you begin

- Install Azure CLI `az` and `kubectl` locally or use Azure CloudShell.

- Create an AKS:

```azurecli-interactive
az aks create \
    --resource-group myResourceGroup \
    --name myAKSCluster \
    --node-count 1 \
    --enable-addons monitoring \
    --kubernetes-version 1.16.7 \
    --generate-ssh-keys \
```

### Built-in storage classes

A storage class is used to define how a unit of storage is dynamically created with a persistent volume. For more information on Kubernetes storage classes, see [Kubernetes Storage Classes](https://kubernetes.io/docs/concepts/storage/storage-classes/).

Each AKS cluster includes two pre-created storage classes, both configured to work with Azure disks:

* The *default* storage class provisions a standard Azure disk.
    * Standard storage is backed by HDDs and delivers cost-effective storage while still being performant. Standard disks are ideal for a cost-effective dev and test workload.
* The *managed-premium* storage class provisions a premium Azure disk.
    * Premium disks are backed by SSD-based high-performance, low-latency disk. Perfect for VMs running production workload. If the AKS nodes in your cluster use premium storage, select the *managed-premium* class.
    
> [!NOTE]

> An Azure disk can only be mounted with *Access mode* type *ReadWriteOnce*, which makes it available to only a single pod in AKS. If you need to share a persistent volume across multiple pods, use *Azure Files*.

> These default storage classes don't allow you to update the volume size once created. 

### Create a persistent volume claim

A persistent volume claim (PVC) is used to automatically provision storage based on a storage class. In this case, a PVC can use one of the pre-created storage classes to create a standard or premium Azure managed disk.

azure-premium.yaml

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: azure-managed-disk
spec:
  accessModes:
  - ReadWriteOnce
  storageClassName: managed-premium
  resources:
    requests:
      storage: 5Gi
```

> [!TIP]
> To create a disk that uses standard storage, use `storageClassName: default` rather than *managed-premium*.

`kubectl apply -f azure-premium.yaml`

### Use the persistent volume

Once the persistent volume claim has been created and the disk successfully provisioned, a pod can be created with access to the disk. The following manifest creates a basic NGINX pod that uses the persistent volume claim named *azure-managed-disk* to mount the Azure disk at the path `/mnt/azure`.

azure-pvc-disk.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        volumeMounts:
        - mountPath: "/mnt/azure"
          name: volume
      volumes:
        - name: volume
          persistentVolumeClaim:
            claimName: azure-managed-disk
```

`kubectl apply -f azure-pvc-disk.yaml`

### Inspect the configuration

This configuration can be seen when inspecting your pod via `kubectl describe pod mypod`, as shown in the following condensed example:

```console
$ kubectl describe pod $POD_NAME

[...]
Volumes:
  volume:
    Type:       PersistentVolumeClaim (a reference to a PersistentVolumeClaim in the same namespace)
    ClaimName:  azure-managed-disk
    ReadOnly:   false
  default-token-smm2n:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  default-token-smm2n
    Optional:    false
[...]
Events:
  Type    Reason                 Age   From                               Message
  ----    ------                 ----  ----                               -------
  Normal  Scheduled              2m    default-scheduler                  Successfully assigned mypod to aks-nodepool1-79590246-0
  Normal  SuccessfulMountVolume  2m    kubelet, aks-nodepool1-79590246-0  MountVolume.SetUp succeeded for volume "default-token-smm2n"
  Normal  SuccessfulMountVolume  1m    kubelet, aks-nodepool1-79590246-0  MountVolume.SetUp succeeded for volume "pvc-faf0f176-8b8d-11e8-923b-deb28c58d242"
[...]
```
