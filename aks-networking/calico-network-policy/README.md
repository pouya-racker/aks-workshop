# Calico-Network-Policy

### Before you begin
- Create AKS cluster and enable Calico network policy.

- Install Azure CLI `az` and `kubectl` locally or use Azure CloudShell.

- Use following command to fetch the AKS cluster credentials

``az aks get-credentials --resource-group RESOURCE_GROUP_NAME --name AKS_CLUSTER_NAME``

### Allow traffic only from within a defined namespace

Create a namespace called development to run the example pods:

`kubectl create namespace development`

`kubectl label namespace/development purpose=development`

Create an example back-end pod that runs NGINX and open port 80 to serve web traffic.
Label the pod with `app=webapp,role=backend` so that we can target it with a network policy in the next section:

`kubectl run backend --image=nginx --labels app=webapp,role=backend --namespace development --expose --port 80 --generator=run-pod/v1`

Create a second namespace to simulate a production namespace:

`kubectl create namespace production`

`kubectl label namespace/production purpose=production`

Schedule a test pod in the production namespace that is labeled as `app=webapp,role=frontend`. 
Attach a terminal session:

`kubectl run --rm -it frontend --image=alpine --labels app=webapp,role=frontend --namespace production --generator=run-pod/v1`

At the shell prompt, use `wget` to confirm that you can access the default NGINX webpage:

`wget -qO- http://backend.development`

Because the labels for the pod match what is currently permitted in the network policy, the traffic is allowed.

Update the network policy, we create a rule using `namespaceSelector` section to only allow traffic from within the development namespace.

kubectl apply -f 

[backend-policy](backend-policy.yaml)

Test the updated network policy, Schedule another pod in the production namespace and attach a terminal session:

`kubectl run --rm -it frontend --image=alpine --labels app=webapp,role=frontend --namespace production --generator=run-pod/v1`

At the shell prompt, use `wget` to see that the network policy now denies traffic:

`wget -qO- --timeout=2 http://backend.development`

With traffic denied from the production namespace, schedule a test pod back in the development namespace and attach a terminal session:

`kubectl run --rm -it frontend --image=alpine --labels app=webapp,role=frontend --namespace development --generator=run-pod/v1`

At the shell prompt, use wget to see that the network policy allows the traffic:

`wget -qO- http://backend`
