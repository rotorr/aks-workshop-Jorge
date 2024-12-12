# Configure TLS for NGINX Ingress Controller

> Estimated Duration: 60 minutes
>> **NOTE:** You need to fulfill these [requirements](environment-setup.md) and [AKS Basic Cluster](aks-basic-cluster.md) to complete this exercise.

## Reference

* [Configure TLS for NGINX Ingress Controller](https://learn.microsoft.com/en-us/azure/aks/csi-secrets-store-nginx-tls)
* [Configure Secrets Stor CSI Driver](https://learn.microsoft.com/en-us/azure/aks/csi-secrets-store-driver)
* [Connect your Azure identity provider to the Azure Key Vault Secrets Store CSI Driver in AKS](https://learn.microsoft.com/en-us/azure/aks/csi-secrets-store-identity-access?tabs=azure-portal&pivots=access-with-a-user-assigned-managed-identity)

## Setup

In a terminal, export variables required for this lab (if not already exported):

```bash
INITIALS=abc
CLUSTER_NAME=aks-$INITIALS
RG=aks-$INITIALS-rg
LOCATION=eastus2
```

If not already connected, connect to the cluster from your local client machine.

```bash
az aks get-credentials --name $CLUSTER_NAME -g $RG
```

## Configure Secrets Store CSI Driver

Upgrade existing AKS cluster with Azure Key Vault provider for Secrets Store CSI Driver capability using these commands. The add-on creates a user-assigned managed identity you can use to authenticate to your key vault.

```bash
az aks enable-addons --addons azure-keyvault-secrets-provider --name $CLUSTER_NAME --resource-group $RG
```

Verify the installation finished successfully, by listing all pods with the `secrets-store-csi-driver` and `secrets-store-provider-azure` labels in the `kube-system` namespace.

```bash
kubectl get pods -n kube-system -l 'app in (secrets-store-csi-driver,secrets-store-provider-azure)'
```

## Create Azure Key Vault

Create a new Azure Key Vault with Azure role-based access control (Azure RBAC) enabled:

```bash
AKV_NAME="$INITIALS-kv"
az keyvault create --name $AKV_NAME --resource-group $RG --location $LOCATION --enable-rbac-authorization
```

Retrieve the Azure Key Vault resource id for later use:

```bash
KEYVAULT_ID=$(az keyvault show --name $AKV_NAME --resource-group $RG \
--query id --output tsv | tr -d '\r')
```

## Connect your Azure identity provider to the Azure Key Vault Secrets Store CSI Driver in AKS

The Secrets Store Container Storage Interface (CSI) Driver on Azure Kubernetes Service (AKS) provides various methods of identity-based access to your Azure Key Vault. In this lab we will use User-assigned managed identity.

Because we are using managed identity we need to additionally enable OIDC issuer and Workload identity capabilities on the AKS cluster:

```bash
az aks update --name $CLUSTER_NAME --resource-group $RG \
  --enable-oidc-issuer --enable-workload-identity
```

### Configure managed identity

Retrieve the user-assigned managed identity created by the add-on. You should also retrieve the identity's `clientId`, which you'll use in later steps when creating a `SecretProviderClass`.

```bash
IDENTITY_CLIENT_ID=$(az aks show -g $RG --name $CLUSTER_NAME --query \
addonProfiles.azureKeyvaultSecretsProvider.identity.clientId -o tsv | tr -d '\r')
```

Create a role assignment that grants the identity permission to access the key vault secrets, access keys, and certificates:

```bash
az role assignment create --role "Key Vault Certificate User" --assignee $IDENTITY_CLIENT_ID --scope $KEYVAULT_ID
```

## Generate a TLS certificate

Generate a TLS certificate using the following command:

```bash
CERT_NAME=aks-ingress-cert
DOMAIN_NAME=$INITIALS-azuredemo.com
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
    -out aks-ingress-tls.crt \
    -keyout aks-ingress-tls.key \
    -subj "/CN=${DOMAIN_NAME}/O=aks-ingress-tls"
```

## Import the certificate into Azure Key Vault

Export the certificate to a PFX file using the following command (skip password prompt):

```bash
openssl pkcs12 -export -in aks-ingress-tls.crt -inkey aks-ingress-tls.key  -out $CERT_NAME.pfx
```

Get the user principal name and the key vault ID:

```bash
PRINCIPAL_NAME=$(az ad signed-in-user show --query userPrincipalName --output tsv | tr -d '\r')
```

Grant Key Vault Contributor role to be able to import certificate:

```bash
az role assignment create --assignee $PRINCIPAL_NAME --role "Key Vault Certificates Officer" --scope $KEYVAULT_ID
```

Import the certificate using this command:

```bash
az keyvault certificate import --vault-name $AKV_NAME --name $CERT_NAME --file $CERT_NAME.pfx
```

## Deploy a SecretProviderClass

Create a namespace:

```bash
NAMESPACE=tls-nginx-ingress
kubectl create namespace $NAMESPACE
```

Retrieve the Azure Keyvault Tenant ID:

```bash
TENANT_ID=$(az keyvault show --name $AKV_NAME --resource-group $RG --query "properties.tenantId" --output tsv | tr -d '\r')
```

Create the `SecretProviderClass` with the certificate name stored in key vault:

```bash
cat <<EOF | kubectl apply -n $NAMESPACE -f -
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: azure-tls
spec:
  provider: azure
  secretObjects:
    - secretName: ingress-tls-csi
      type: kubernetes.io/tls
      data:
        - objectName: $CERT_NAME
          key: tls.key
        - objectName: $CERT_NAME
          key: tls.crt
  parameters:
    usePodIdentity: "false"
    useVMManagedIdentity: "true"
    userAssignedIdentityID: $IDENTITY_CLIENT_ID
    keyvaultName: $AKV_NAME
    objects: |
      array:
        - |
          objectName: $CERT_NAME
          objectType: secret
    tenantId: $TENANT_ID
EOF
```

## Configure and deploy the NGINX ingress controller

Add the official `ingress-nginx` chart repository using the following `helm` commands:

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
```

Using helm, install the `ingress-nginx` chart, passing the [manifests/nginx-tls.values] files to configure the controller to use the `azure-tls` secret provider class.

```bash
helm install ingress-nginx/ingress-nginx --generate-name \
    --namespace $NAMESPACE \
    --set controller.replicaCount=2 \
    --set controller.nodeSelector."kubernetes\.io/os"=linux \
    --set defaultBackend.nodeSelector."kubernetes\.io/os"=linux \
    --set controller.service.annotations."service\.beta\.kubernetes\.io/azure-load-balancer-health-probe-request-path"=/healthz \
    -f manifests/nginx-tls.values
```

Verify that the `ingress-tls-csi` secret was created:

```bash
kubectl get secret ingress-tls-csi -n $NAMESPACE -o yaml
```

## Deploy the application

Deploy application using this command:

```bash
kubectl apply -f manifests/aks-helloworld.yaml -n $NAMESPACE
```

Run this command to confirm pod is deployed and in Running Status:

```bash
kubectl get all -n $NAMESPACE
```

Next deploy ingress configured to use `ingress-tls-csi` secret:

```bash
cat <<EOF | kubectl apply -n $NAMESPACE -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-tls
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/rewrite-target: /$2
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - $DOMAIN_NAME
    secretName: ingress-tls-csi
  rules:
  - host: $DOMAIN_NAME
    http:
      paths:
      - path: /hello-world(/|$)(.*)
        pathType: Prefix
        backend:
          service:
            name: aks-helloworld
            port:
              number: 80
      - path: /(.*)
        pathType: Prefix      
        backend:
          service:
            name: aks-helloworld
            port:
              number: 80
EOF
```

Get the external IP for the `nginx-ingress` service (re-run command until IP value is populated):

```bash
EXTERNAL_IP=$(kubectl get service -n $NAMESPACE -l app.kubernetes.io/name=ingress-nginx -o jsonpath='{.items[0].status.loadBalancer.ingress[0].ip}')
```

Verify your ingress is properly configured with TS using the following command:

```bash
curl -v -k --resolve $DOMAIN_NAME:443:$EXTERNAL_IP https://$DOMAIN_NAME
```

You should see the server certificate in the output

## Cleanup

Once done testing, uninstall the helm chart and delete the namespace:

```bash
RELEASE_NAME=$(helm list -n $NAMESPACE -o json | jq -r '.[0].name')
helm uninstall $RELEASE_NAME -n $NAMESPACE
kubectl delete namespace $NAMESPACE
```
