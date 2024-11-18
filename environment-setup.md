# Environment Setup

The following are pre-requisites before starting the labs:

* Azure CLI
* kubectl
* Shell Terminal: e.g. git-bash, WSL

Login to Azure.

```bash
az login
```

Install the following Azure CLI extensions:

```bash
az extension add -n ssh
```

Register the following providers:

```bash
az provider register --namespace Microsoft.Storage
az provider register --namespace Microsoft.Compute
az provider register --namespace Microsoft.Network
az provider register --namespace Microsoft.Monitor
az provider register --namespace Microsoft.ManagedIdentity
az provider register --namespace Microsoft.KeyVault
az provider register --namespace Microsoft.Kubernetes
```
