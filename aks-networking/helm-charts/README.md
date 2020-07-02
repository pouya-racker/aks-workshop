# Helm-Charts

### Before you begin
Make sure you have the Helm CLI installed. See [Installing Helm](https://helm.sh/docs/intro/install/)

### Verify Helm version
Make sure you have Helm 3 installed.
`helm version`

### Install an application with Helm v3
Use the helm repo command to add the official Helm stable charts repository.

`helm repo add stable https://kubernetes-charts.storage.googleapis.com/`

Find Helm charts

`helm search repo stable`

To update the list of charts, use the helm repo update command.

`helm repo update`

Run helm charts:

`helm install my-nginx-ingress stable/nginx-ingress --set controller.nodeSelector."beta\.kubernetes\.io/os"=linux --set defaultBackend.nodeSelector."beta\.kubernetes\.io/os"=linux`
    
List releases:

`helm list`

Cleanup:

`helm uninstall my-nginx-ingress`
