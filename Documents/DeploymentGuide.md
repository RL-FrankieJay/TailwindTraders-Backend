# Deploy Backend services on AKS

Pre-requisites for this deployment are to have 

* The AKS and all related resources deployed in Azure  
* A terminal with
    * Bash environment with [jq](https://stedolan.github.io/jq/) installed **-OR-**
    * Powershell environment
* [Azure CLI 2.0](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest) installed.
* [Kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/) installed.
* Docker installed

**Note**: The easiest way to have a working Bash environment on Windows is [enabling the WSL](https://docs.microsoft.com/en-us/windows/wsl/install-win10) and installing a Linux distro from the Windows Store.

## Connecting kubectl to AKS

From the terminal type:

* `az login` and follow instructions to log into your Azure.
* If you have more than one subscription type `az account list -o table` to list all your Azure subscriptions. Then type  `az account set --subscription <subscription-id>` to select your subscription
* `az aks get-credentials -n <your-aks-name> -g <resource-group-name>` to download the configuration files that `kubectl` needs to connect to your AKS.

At this point if you type `kubectl config current-context` the name of your AKS cluster should be displayed. That means that `kubectl` is ready to use your AKS

## Installing Tiller on AKS

Helm is a tool to deploy resources in a Kubernetes cluster in a clean and simple manner. It is composed of two tools, one client-side (the Helm client) that needs to be installed on your machine, and a server component called _Tiller_ that has to be installed on the Kubernetes cluster.

To install Helm, refer to its [installation page](https://docs.helm.sh/using_helm/#installing-helm). Once Helm is installed, _Tiller_ must be deployed on the cluster. For deploying _Tiller_ run the `add-tiller.sh` (from Bash) or the `Add-Tiller.ps1` (from Powershell).

Once installed, helm commands like `helm ls` should work without any error.

## Configuring services

Before deploying services using Helm, you need to setup the configuration by editing the file `helm/gvalues.yaml` and put the secrets, connection strings and all the configuration.

Please refer to the comments of the file for its usage. Just ignore (but not delete) the `tls` section as, currently, TLS is not supported (its on the roadmap).

## Create secrets on the AKS

Docker images are stored in a ACR (a private Docker Registry hosted in Azure).

Before deploying anything on AKS, a secret must be installed to allow AKS to connect to the ACR through a Kubernetes' service account. 

To do so from a Bash terminal run the file `./create-secret.sh` with following parameters:

* `-g <group>` Resource group where AKS is
* `--acr-name <name>`  Name of the ACR
* `--clientid <id>` Client id of the service principal to use
* `--password <pwd>` Service principal password

If using Powershell run the `./Create-Secret.ps1` with following parameters:

* `-resourceGroup <group>` Resource group where AKS is
* `-acrName <name>`  Name of the ACR
* `-clientId <id>` Client id of the service principal to use
* `-password <pwd>` Service principal password

Please, note that the Service principal must be already exist. To create a service principal you can run the command `az ad sp create-for-rbac`.

## Build & deploy images to ACR

Run `docker-compose build` and then manually tag and push the images to your ACR.

**Note**: Under WSL docker daemon do not run, so you won't be able to use `docker-compose build` unless you configure docker client to use the Docker CE for Windows daemon. You can always build docker images using Docker CE for Windows, from a Windows command prompt, and run all scripts from WSL.

## Deploying services

>**Note**: If you want to add SSL/TLS support on the cluster (needed to use https on the web) plase read following section **before installing the backend**.

To deploy the services from a Bash terminal run the `./deploy-images-aks.sh` script with the following parameters:

* `-n <name>` Name of the deployment. Defaults to  `my-tt`
* `--aks-name <name>` Name of the AKS
* `-g <group>` Name of the resource group
* `--acr-name <name>` Name of the ACR
* `--tag <tag>` Docker images tag to use. Defaults to  `latest`
* `--charts <charts>` List of comma-separated values with charts to install. Defaults to `*` (all)

If using Powershell, have to run `./Deploy-Images-Aks.ps1` with following parameters:

* `-name <name>` Name of the deployment. Defaults to  `my-tt`
* `-aksName <name>` Name of the AKS
* `-resourceGroup <group>` Name of the resource group
* `-acrName <name>` Name of the ACR
* `-tag <tag>` Docker images tag to use. Defaults to  `latest`
* `-charts <charts>` List of comma-separated values with charts to install. Defaults to `*` (all)
* `-tlsEnv prod|staging` If **SSL/TLS support has been installed**, you have to use this parameter to enable https endpoints. Value must be `staging` or `prod` and must be the same value used when you installed SSL/TLS support. If SSL/TLS is not installed, you can omit this parameter.

This script will install all services using Helm and your custom configuration from file `gvalues.yaml`

The parameter `charts` allow for a selective installation of charts. Is a list of comma-separated values that mandates the services to deploy in the AKS. Values are:

* `pr` Products API
* `cp` Coupons API
* `pf` Profiles API
* `pp` Popular products API
* `st` Stock API
* `ic` Image classifier API
* `ct` Shopping cart API
* `mgw` Mobile Api Gateway
* `wgw` Web Api Gateway

So, using `charts pp,st` will only install the popular products and the stock api.

## Enabling SSL/TLS on the cluster

SSL/TLS support is provided by [cert-manager](https://github.com/jetstack/cert-manager) that allows auto-provisioning of TLS certificates using [Let's Encrypt](https://letsencrypt.org/) and [ACME](https://en.wikipedia.org/wiki/Automated_Certificate_Management_Environment) protocol. 


To enable SSL/TLS support you must do it **before deploying your images**. The first step is to add cert-manager to the cluster by running `./add-cert-manager.sh` or `./Add-Cert-Manager.ps1`. Both scripts accept no parameters and they use helm to configure cert-manager in the cluster. **This needs to be done only once**

Then you should run `./Enable-Ssl.ps1` with following parameters:

* `sslSupport`: Use `staging` or `prod` to use the staging or production environments of Let's Encrypt
* `aksName`: The name of the AKS to use
* `resourceGroup`: Name of the resource group where AKS is
* `domain`: Domain to use for the SSL/TLS certificates. Is **optional** and if not used it defaults to the public domain of the AKS. Only need to use this parameter if using custom domains

Output of the script will be something like following:

``` 
NAME:   my-tt-ssl
LAST DEPLOYED: Fri Dec 21 11:32:00 2018
NAMESPACE: default
STATUS: DEPLOYED

RESOURCES:
==> v1alpha1/Certificate
NAME             AGE
tt-cert-staging  0s

==> v1alpha1/Issuer
NAME                 AGE
letsencrypt-staging  0s
```

You can verify that the _issuer_ object is created using `kubectl get issuers`:

```
PS> kubectl get issuers
NAME                  AGE
letsencrypt-staging   4m
```

You can verify that the _certificate_ object is created using `kubectl get certificates`:

```
PS> kubectl get certificates
NAME              AGE
tt-cert-staging   4m
```

The _certificate_ object is not the real SSL/TLS certificate but a definition on how get one from Let's Encrypt. The certificate itself is stored in a secret, called `letsencrypt-staging` (or `letsencrypt-prod`). You should see a secret named `tt-letsencrypt-xxxx` (where `xxxx` is either `staging` or `prod`).

```
PS> kubectl get secrets
NAME                  TYPE                                  DATA      AGE
acr-auth              kubernetes.io/dockerconfigjson        1         2d
default-token-6tm9t   kubernetes.io/service-account-token   3         3d
letsencrypt-prod      Opaque                                1         3h
letsencrypt-staging   Opaque                                1         4h
tt-letsencrypt-prod   kubernetes.io/tls                     2         5m
ttsa-token-rkjlg      kubernetes.io/service-account-token   3         2d
```

The SSL/TLS secret names are:

* `letsencrypt-staging`: Secret for the staging _issuer_. This is NOT the SSL/TLS certificate
* `tt-letsencrypt-staging`: Secret for the staging SSL/TLS certificate.
* `letsencrypt-prod`: Secret for the prod _issuer_. This is NOT the SSL/TLS certificate
* `tt-letsencrypt-prod`: Secret for the prod SSL/TLS certificate.

At this point **the support for SSL/TLS is installed, and you can install Tailwind Traders Backend on the repo**.

>**Note:** You don't need to do this again, unless you want to change the domain of the SSL/TLS certificate. In this case you need to remove the issuer and certificate objects (using `helm delete my-tt-ssl --purge` and then reinstall again)

>**Note** Staging certificates **are not trust**, so browsers will complain about it, exactly in the same way that they complain about a self-signed certificate. The only purpose is to test all the deployment works, but in any production environment you must use the `prod` environment. Main difference is the Let's Encrypt API call rates are more limited than the staging ones.

Another way to validate your certificate deployment is doing a `kubectl describe cert tt-cert-staging` (or `tt-cert-prod`). In the `Events` section you should see that the certificate has been obtained:

```
Events:
  Type    Reason          Age   From          Message
  ----    ------          ----  ----          -------
  Normal  CreateOrder     10m   cert-manager  Created new ACME order, attempting validation...
  Normal  DomainVerified  9m    cert-manager  Domain "e43cd6ae16f344a093dc.eastus.aksapp.io" verified with "http-01" validation
  Normal  IssueCert       9m    cert-manager  Issuing certificate...
  Normal  CertObtained    9m    cert-manager  Obtained certificate from ACME server
  Normal  CertIssued      9m    cert-manager  Certificate issued successfully
```