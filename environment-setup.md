# Environment Setup

## Prior to Lab Day

The following are pre-requisites before starting the labs:

* [Azure CLI](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli)
* [kubectl](https://kubernetes.io/docs/tasks/tools/#kubectl)
* [Helm](https://helm.sh/docs/intro/install/)
* Shell Terminal: e.g. [git-bash](https://git-scm.com/downloads), [WSL](https://learn.microsoft.com/en-us/windows/wsl/install) or Mac OS Terminal
* [Certbot](https://eff-certbot.readthedocs.io/en/stable/install.html#installation)
  * Option 1: using WSL, install using: `sudo apt-get install certbot`
  * Option 2: using Docker command:
        ```
        docker run -it --rm --name certbot \
        -v "/etc/letsencrypt:/etc/letsencrypt" \
        -v "/var/lib/letsencrypt:/var/lib/letsencrypt" \
        certbot/certbot certonly
        ```

## During Lab Day

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
