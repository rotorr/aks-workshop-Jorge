# Environment Setup

## Prior to Lab Day

### Option 1: Using Github Codespaces (recommended)

#### Create a GitHub account

1. If you don't already have access to a GitHub account, navigate to: [https://github.com](https://github.com)
1. Click on sign up.
1. Use an email of your choice to create an account.
1. Login to your new account.

#### Create a fork of this GitHub Repository and create Github codespace

1. Navigate to this link to create a new [fork](https://github.com/yortch/aks-workshop/fork) (must be logged into your github account).
1. Accept the default values and click on **"Create fork"** which will take you to the forked repository in the browser.
1. From your forked repository click on the **"<> Code"** button. Then click on the **"Create codespace on main"** button.

### Option 2: Using your own computer

The following are pre-requisites before starting the labs:

* Linux Shell Terminal
  * Windows options: [git-bash](https://git-scm.com/downloads) or [WSL](https://learn.microsoft.com/en-us/windows/wsl/install)
  * Mac OS Terminal
* [Azure CLI](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli)
* [kubectl](https://kubernetes.io/docs/tasks/tools/#kubectl)
* [Helm](https://helm.sh/docs/intro/install/)
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
