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
```

### Run the voting app

The sample application used in this tutorial is a basic voting app. 
The application consists of a front-end web component and a back-end Redis instance. 
The web component is packaged into a custom container image. 
The Redis instance uses an unmodified image from Docker Hub.
Deploy the app using:

`kubectl apply -f voting-app.yaml`

Voting app is exposed publicly using a LoadBalancer service, retrieve the LB's public IP and browse the app.

`kubectl get svc`  

### Manually scale pods

When the Azure Vote front-end and Redis instance were deployed in previous tutorials, a single replica was created.
To manually change the number of pods in the azure-vote-front deployment, use the `kubectl scale` command. The following example increases the number of front-end pods to 5: 

`kubectl scale --replicas=5 deployment/azure-vote-front`

Run `kubectl get pods` again to verify that AKS creates the additional pods.

### Autoscale pods

Kubernetes supports [horizontal pod autoscaling](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/) to adjust the number of pods in a deployment depending on CPU utilization or other select metrics. The [Metrics Server](https://kubernetes.io/docs/tasks/debug-application-cluster/resource-metrics-pipeline/#metrics-server) is used to provide resource utilization to Kubernetes, and is automatically deployed in AKS clusters versions 1.10 and higher.

To use the autoscaler, all containers in your pods and your pods must have CPU requests and limits defined. In the `azure-vote-front` deployment, the front-end container already requests 0.25 CPU, with a limit of 0.5 CPU. These resource requests and limits are defined as shown in the following example snippet:

```yaml
resources:
  requests:
     cpu: 250m
  limits:
     cpu: 500m
```

The following example uses the `kubectl autoscale` command to autoscale the number of pods in the *azure-vote-front* deployment. If average CPU utilization across all pods exceeds 50% of their requested usage, the autoscaler increases the pods up to a maximum of *10* instances. A minimum of *3* instances is then defined for the deployment:

```console
kubectl autoscale deployment azure-vote-front --cpu-percent=50 --min=3 --max=10
```

Alternatively, you can create a manifest file to define the autoscaler behavior and resource limits. The following is an example of a manifest file named `azure-vote-hpa.yaml`.

```yaml
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: azure-vote-back-hpa
spec:
  maxReplicas: 10 # define max replica count
  minReplicas: 3  # define min replica count
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: azure-vote-back
  targetCPUUtilizationPercentage: 50 # target CPU utilization

---

apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: azure-vote-front-hpa
spec:
  maxReplicas: 10 # define max replica count
  minReplicas: 3  # define min replica count
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: azure-vote-front
  targetCPUUtilizationPercentage: 50 # target CPU utilization
```

Use `kubectl apply` to apply the autoscaler defined in the `azure-vote-hpa.yaml` manifest file.

```
kubectl apply -f azure-vote-hpa.yaml
```

To see the status of the autoscaler, use the `kubectl get hpa` command as follows:

```
kubectl get hpa

NAME               REFERENCE                     TARGETS    MINPODS   MAXPODS   REPLICAS   AGE
azure-vote-front   Deployment/azure-vote-front   0% / 50%   3         10        3          2m
```

After a few minutes, with minimal load on the Azure Vote app, the number of pod replicas decreases automatically to three. You can use `kubectl get pods` again to see the unneeded pods being removed.