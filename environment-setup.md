# Environment Setup

The following are pre-requisites before starting the labs:

* [Azure CLI](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli)
* [kubectl](az aks install-cli)
* Shell Terminal: e.g. [git-bash](https://git-scm.com/downloads), [WSL](https://learn.microsoft.com/en-us/windows/wsl/install) or MacOS Terminal

Login to Azure.

```bash
az login
```

Install the following Azure CLI extensions:

```bash
az extension add -n ssh
az extension add -n azure-firewall
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
