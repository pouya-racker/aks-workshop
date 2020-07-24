# AKS Identity & Access

## AKS-managed Azure Active Directory integration

AKS-managed Azure AD integration is designed to simplify the Azure AD integration experience, where users were previously required to create a client app, a server app, and required the Azure AD tenant to grant Directory Read permissions. In the new version, the AKS resource provider manages the client and server apps for you.
For reviewing the legacy method see [azure-ad-integration-cli](https://docs.microsoft.com/en-us/azure/aks/azure-ad-integration-cli) 

### Prerequisites

* The Azure CLI version 2.9.0 or later
* Kubectl with a minimum version of [1.18](https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG/CHANGELOG-1.18.md#v1180)

### Before you begin

For your cluster, you need an Azure AD group. This group is needed as admin group for the cluster to grant cluster admin permissions. You can use an existing Azure AD group, or create a new one.

```azurecli-interactive
# List existing groups in the directory
az ad group list --filter "displayname eq '<group-name>'" -o table
```

To create a new Azure AD group for your cluster administrators, use the following command:

```azurecli-interactive
# Create an Azure AD group
az ad group create --display-name myAKSAdminGroup --mail-nickname myAKSAdminGroup
```

### Create an AKS cluster with Azure AD enabled

Create an AKS cluster by using the following CLI commands.

Create an Azure resource group:

```azurecli-interactive
# Create an Azure resource group
az group create --name myResourceGroup --location westus
```

Create an AKS cluster, and enable administration access for your Azure AD group

```azurecli-interactive
# Create an AKS-managed Azure AD cluster
az aks create -g myResourceGroup -n myAKSCluster --node-count 1 --enable-aad --aad-admin-group-object-ids <id> --aad-tenant-id <id>
```

### Access an Azure AD enabled cluster

You'll need the [Azure Kubernetes Service Cluster User](https://docs.microsoft.com/en-us/azure/role-based-access-control/built-in-roles#azure-kubernetes-service-cluster-user-role) built-in role to do the following steps.

Get the user credentials to access the cluster:
 
```azurecli-interactive
 az aks get-credentials --resource-group myResourceGroup --name myManagedCluster
```
Follow the instructions to sign in.

Use the kubectl get nodes command to view nodes in the cluster:

```azurecli-interactive
kubectl get nodes
```

## Using AKS RBAC

### Create demo groups and user in Azure AD

In this demo, let's create a user roles that can be used to show how Kubernetes RBAC and Azure AD control access to cluster resources. The following example role is used:

**Application developer**
  * A user named *aksdev* that is part of the *appdev* group.

In production environments, you can use existing users and groups within an Azure AD tenant.

First, get the resource ID of your AKS cluster using the `az aks show` command. Assign the resource ID to a variable named *AKS_ID* so that it can be referenced in additional commands.

```azurecli-interactive
AKS_ID=$(az aks show \
    --resource-group myResourceGroup \
    --name myAKSCluster \
    --query id -o tsv)
```

Create the example group in Azure AD for the application developers using the `az ad group create` command. The following example creates a group named *appdev*:

```azurecli-interactive
APPDEV_ID=$(az ad group create --display-name appdev --mail-nickname appdev --query objectId -o tsv)
```

Now, create an Azure role assignment for the *appdev* group using the `az role assignment create` command. This assignment lets any member of the group use `kubectl` to interact with an AKS cluster by granting them the *Azure Kubernetes Service Cluster User Role*.

```azurecli-interactive
az role assignment create \
  --assignee $APPDEV_ID \
  --role "Azure Kubernetes Service Cluster User Role" \
  --scope $AKS_ID
```

Now lets create an example users. To test the RBAC integration at the end of the demo, you sign in to the AKS cluster with this account.

The following example creates a user with the display name *AKS Dev* and the user principal name (UPN) of `aksdev@contoso.com`. Update the UPN to include a verified domain for your Azure AD tenant (replace *contoso.com* with your own domain), and provide your own secure `--password` credential:

```azurecli-interactive
AKSDEV_ID=$(az ad user create \
  --display-name "AKS Dev" \
  --user-principal-name aksdev@contoso.com \
  --password P@ssw0rd1 \
  --query objectId -o tsv)
```

Now add the user to the *appdev* group created in the previous section using the `az ad group member add` command:

```azurecli-interactive
az ad group member add --group appdev --member-id $AKSDEV_ID
```

### Create the AKS resources for app devs

First, get the cluster admin credentials using the `az aks get-credentials` command. In one of the following sections, you get the regular *user* cluster credentials to see the Azure AD authentication flow in action.

```azurecli-interactive
az aks get-credentials --resource-group myResourceGroup --name myAKSCluster --admin
```

Create a namespace in the AKS cluster using the `kubectl create namespace` command. The following example creates a namespace name *dev*:

```console
kubectl create namespace dev
```

In Kubernetes, *Roles* define the permissions to grant, and *RoleBindings* apply them to desired users or groups. These assignments can be applied to a given namespace, or across the entire cluster. For more information, see [Using RBAC authorization](https://docs.microsoft.com/en-us/azure/aks/concepts-identity#kubernetes-role-based-access-controls-rbac).

First, create a Role for the *dev* namespace. This role grants full permissions to the namespace. In production environments, you can specify more granular permissions for different users or groups.

Create a file named `role-dev-namespace.yaml` and paste the following YAML manifest:

```yaml
kind: Role
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: dev-user-full-access
  namespace: dev
rules:
- apiGroups: ["", "extensions", "apps"]
  resources: ["*"]
  verbs: ["*"]
- apiGroups: ["batch"]
  resources:
  - jobs
  - cronjobs
  verbs: ["*"]
```

Create the Role using the `kubectl apply`.

```console
kubectl apply -f role-dev-namespace.yaml
```

Next, get the resource ID for the *appdev* group using the `az ad group show` command. This group is set as the subject of a RoleBinding in the next step.

```azurecli-interactive
az ad group show --group appdev --query objectId -o tsv
```

Now, create a RoleBinding for the *appdev* group to use the previously created Role for namespace access. Create a file named `rolebinding-dev-namespace.yaml` and paste the following YAML manifest. On the last line, replace *groupObjectId*  with the group object ID output from the previous command:

```yaml
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: dev-user-access
  namespace: dev
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: dev-user-full-access
subjects:
- kind: Group
  namespace: dev
  name: groupObjectId
```

Create the RoleBinding using the `kubectl apply` command and specify the filename of your YAML manifest:

```console
kubectl apply -f rolebinding-dev-namespace.yaml
```

### Interact with cluster resources using Azure AD identities

Now, let's test the expected permissions work when you create and manage resources in an AKS cluster. In these examples, you schedule and view pods in the user's assigned namespace. Then, you try to schedule and view pods outside of the assigned namespace.

First, reset the *kubeconfig* context using the `az aks get-credentials` command. In a previous section, you set the context using the cluster admin credentials. The admin user bypasses Azure AD sign in prompts. Without the `--admin` parameter, the user context is applied that requires all requests to be authenticated using Azure AD.

```azurecli-interactive
az aks get-credentials --resource-group myResourceGroup --name myAKSCluster --overwrite-existing
```

Schedule a basic NGINX pod using the `kubectl run` command in the *dev* namespace:

```console
kubectl run nginx-dev --image=nginx --namespace dev
```

As the sign in prompt, enter the credentials for your own `appdev@contoso.com` account created at the start of the demo. Once you are successfully signed in, the account token is cached for future `kubectl` commands. The NGINX is successfully schedule, as shown in the following example output:

```console
$ kubectl run nginx-dev --image=nginx --namespace dev

To sign in, use a web browser to open the page https://microsoft.com/devicelogin and enter the code B24ZD6FP8 to authenticate.

pod/nginx-dev created
```

Now use the `kubectl get pods` command to view pods in the *dev* namespace.

```console
kubectl get pods --namespace dev
```

As shown in the following example output, the NGINX pod is successfully *Running*:

```console
$ kubectl get pods --namespace dev

NAME        READY   STATUS    RESTARTS   AGE
nginx-dev   1/1     Running   0          4m
```

### Create and view cluster resources outside of the assigned namespace

Now try to view pods outside of the *dev* namespace. Use the `kubectl get pods` command again, this time to see `--all-namespaces` as follows:

```console
kubectl get pods --all-namespaces
```

The user's group membership does not have a Kubernetes Role that allows this action, as shown in the following example output:

```console
$ kubectl get pods --all-namespaces

Error from server (Forbidden): pods is forbidden: User "aksdev@contoso.com" cannot list resource "pods" in API group "" at the cluster scope
```
