# TAP with Flux on AKS

This is an example implementation of installing TAP on AKS using Flux. In this case Flux will be provided by TMC and we will be usign TMC cluster groups for some flux bootstrapping.

## Pre-reqs

* 3 AKS clusters(view,build,run)
* azure cli
* azwi cli
  



# Setup

## Create cluster groups

This example will not walk through creating the cluster groups or cluster but here is the setup that this is tested with. Cluster groups with clusters shown below

* tap-flux-core
  * tap-flux-build
  * tap-flux-view
* tap-dev-env
  * tap-flux-dev-run

## enable OIDC issuer 

this may already be done depending how how the clusters were created. if they were created through TMC AKS LCM. run the below command on each cluster.

```bash
az aks update -g myResourceGroup -n myAKSCluster --enable-oidc-issuer 
```

## Create an AKV to store our secrets

We are going to use role based access for AKV with external secrets operator. we need to have a role created. 


```bash
export KEYVAULT_NAME="tap-flux-core-kv"
export RESOURCE_GROUP="tap-flux-kv-group"
export LOCATION="westus2"
```

```bash
az group create --name "${RESOURCE_GROUP}" --location "${LOCATION}"
```

```bash
az keyvault create --resource-group "${RESOURCE_GROUP}" \
   --location "${LOCATION}" \
   --name "${KEYVAULT_NAME}"

```

## Create an AAD application & grant permissions

```bash
export APPLICATION_NAME="eso-flux-tap"
```

```bash
azwi serviceaccount create phase app --aad-application-name "${APPLICATION_NAME}"
```

```bash
export APPLICATION_CLIENT_ID="$(az ad sp list --display-name "${APPLICATION_NAME}" --query '[0].appId' -otsv)"
az keyvault set-policy --name "${KEYVAULT_NAME}" \
  --secret-permissions get \
  --spn "${APPLICATION_CLIENT_ID}"
```


## Federate the k8s service account with Azure

this needs to be run with each clusters issuer url

```bash
export CLUSTER_NAME="cluster-name"
export CLUSTER_RG="cluster-resource-group"
export SERVICE_ACCOUNT_ISSUER=$(az aks show --resource-group $CLUSTER_RG --name $CLUSTER_NAME --query "oidcIssuerProfile.issuerUrl" -otsv)
export SERVICE_ACCOUNT_NAMESPACE="external-secrets"
export SERVICE_ACCOUNT_NAME="akv-reader"

azwi serviceaccount create phase federated-identity \
  --aad-application-name "${APPLICATION_NAME}" \
  --service-account-namespace "${SERVICE_ACCOUNT_NAMESPACE}" \
  --service-account-name "${SERVICE_ACCOUNT_NAME}" \
  --service-account-issuer-url "${SERVICE_ACCOUNT_ISSUER}"

```

## Enable gitops on the cluster groups for your clusters

