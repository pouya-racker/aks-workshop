# NGINX-Ingress-Controller

### Before you begin
-Make sure you have the Helm CLI installed or user Azure CloudShell. See [Installing Helm](https://helm.sh/docs/intro/install/),
We will use Helm 3 to install NGINX ingress controller.

- Create AKS cluster.

- Install Azure CLI `az` and `kubectl` locally or use Azure CloudShell.

- Use following command to fetch the AKS cluster credentials

``az aks get-credentials --resource-group RESOURCE_GROUP_NAME --name AKS_CLUSTER_NAME``

### Installation
Create an ingress controller namespace

`kubectl create namespace ingress-basic`

Add the official stable repository

`helm repo add stable https://kubernetes-charts.storage.googleapis.com/`

Use Helm to deploy an NGINX ingress controller

```
helm install nginx-ingress stable/nginx-ingress \
    --namespace ingress-basic \
    --set controller.replicaCount=2 \
    --set controller.nodeSelector."beta\.kubernetes\.io/os"=linux \
    --set defaultBackend.nodeSelector."beta\.kubernetes\.io/os"=linux
```
    
**Tips:**

- For added redundancy, two replicas of the NGINX ingress controllers are deployed with the `--set controller.replicaCount` parameter. 
To fully benefit from running replicas of the ingress controller, make sure there's more than one node in your AKS cluster.

- The ingress controller also needs to be scheduled on a Linux node. Windows Server nodes shouldn't run the ingress controller. 
A node selector is specified using the `--set nodeSelector` parameter to tell the Kubernetes scheduler to run the NGINX ingress controller on a Linux-based node.

- If you would like to enable client source IP preservation for requests to containers in your cluster, add `--set controller.service.externalTrafficPolicy=Local` to the Helm install command. 
The client source IP is stored in the request header under X-Forwarded-For.

**Create demo application and Ingress resource**

`kubectl apply -f`

[demo-app-1](aks-helloworld-one.yaml)

[demo-app-1](aks-helloworld-two.yaml)

[ingress](aks-helloworld-ingress.yaml)

### HTTPS Ingress with TLS certificate
The following example generates a 2048-bit RSA X509 certificate valid for 365 days named aks-ingress-tls.crt. 
The private key file is named aks-ingress-tls.key. A Kubernetes TLS secret requires both of these files.

```
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
    -out aks-ingress-tls.crt \
    -keyout aks-ingress-tls.key \
    -subj "/CN=demo.azure.com/O=aks-ingress-tls"
```

Create Kubernetes Secret for the TLS certification

```
kubectl create secret tls aks-ingress-tls \
    --namespace ingress-basic \
    --key aks-ingress-tls.key \
    --cert aks-ingress-tls.crt
```

`kubectl apply -f`

[tls-ingress](aks-helloworld-tls-ingress.yaml)

Test the ingress configuration

Use curl and specify the `--resolve` parameter. This parameter lets you map the demo.azure.com name to the public IP address of your ingress controller. 
Specify the public IP address of your own ingress controller

`curl -v -k --resolve demo.azure.com:$PUBLIC_IP https://demo.azure.com`

For using ingress with internal loadbalancer see: [ingress-internal-ip](https://docs.microsoft.com/en-us/azure/aks/ingress-internal-ip)