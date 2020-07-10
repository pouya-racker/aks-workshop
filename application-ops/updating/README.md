# Update an AKS application

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

### Deploy the updated application

To provide maximum uptime, multiple instances of the application pod must be running. Verify the number of running front-end instances with the `ubectl get pods` command:

```
$ kubectl get pods

NAME                               READY     STATUS    RESTARTS   AGE
azure-vote-back-217588096-5w632    1/1       Running   0          10m
azure-vote-front-233282510-b5pkz   1/1       Running   0          10m
azure-vote-front-233282510-dhrtr   1/1       Running   0          10m
azure-vote-front-233282510-pqbfk   1/1       Running   0          10m
```

If you don't have multiple front-end pods, scale the *azure-vote-front* deployment as follows:

```console
kubectl scale --replicas=3 deployment/azure-vote-front
```

To update the application, use the `kubectl set` command. 

```console
kubectl set image deployment azure-vote-front azure-vote-front=microsoft/azure-vote-front:v2
```

To monitor the deployment, use the `kubectl get pod` command. As the updated application is deployed, your pods are terminated and re-created with the new container image.

```console
kubectl get pods
```

The following example output shows pods terminating and new instances running as the deployment progresses:

To see the rollout status run:

`kubectl rollout status deployment azure-vote-front`

```
$ kubectl get pods

NAME                               READY     STATUS        RESTARTS   AGE
azure-vote-back-2978095810-gq9g0   1/1       Running       0          5m
azure-vote-front-1297194256-tpjlg  1/1       Running       0          1m
azure-vote-front-1297194256-tptnx  1/1       Running       0          5m
azure-vote-front-1297194256-zktw9  1/1       Terminating   0          1m
```

### Test the updated application

To view the update application, first get the external IP address of the `azure-vote-front` service:

```console
kubectl get service azure-vote-front
```

Now open a local web browser to the IP address of your service.

###Rolling Back to a Previous Revision

Follow the steps given below to rollback the Deployment from the current version to the previous version, which is version 1.

`kubectl rollout undo deployment azure-vote-front`

The output is similar to this:

`deployment.apps/azure-vote-front rolled back`