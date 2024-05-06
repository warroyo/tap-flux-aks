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

## Create a AAD workload identity for external DNS

```bash
export APPLICATION_NAME="dns-flux-tap"
```

```bash
azwi serviceaccount create phase app --aad-application-name "${APPLICATION_NAME}"
```

```bash
export APPLICATION_CLIENT_ID="$(az ad sp list --display-name "${APPLICATION_NAME}" --query '[0].appId' -otsv)"
export AZURE_DNS_ZONE_RESOURCE_GROUP="main-dns"
export AZURE_DNS_ZONE="azure.warroyo.com"
export DNS_ID=$(az network dns zone show --name "${AZURE_DNS_ZONE}" \
  --resource-group "${AZURE_DNS_ZONE_RESOURCE_GROUP}" --query "id" --output tsv)
export RESOURCE_GROUP_ID=$(az group show --name "${AZURE_DNS_ZONE_RESOURCE_GROUP}" --query "id" --output tsv)
```

```bash
az role assignment create --role "DNS Zone Contributor" \
  --assignee "${APPLICATION_CLIENT_ID}" --scope "${DNS_ID}"

az role assignment create --role "Reader" \
  --assignee "${APPLICATION_CLIENT_ID}" --scope "${RESOURCE_GROUP_ID}"

```  


## federate the new app with the k8s service account

this needs to be run with each clusters issuer url

```bash
export CLUSTER_NAME="cluster-name"
export CLUSTER_RG="cluster-resource-group"
export SERVICE_ACCOUNT_ISSUER=$(az aks show --resource-group $CLUSTER_RG --name $CLUSTER_NAME --query "oidcIssuerProfile.issuerUrl" -otsv)
export SERVICE_ACCOUNT_NAMESPACE="external-dns"
export SERVICE_ACCOUNT_NAME="default"

azwi serviceaccount create phase federated-identity \
  --aad-application-name "${APPLICATION_NAME}" \
  --service-account-namespace "${SERVICE_ACCOUNT_NAMESPACE}" \
  --service-account-name "${SERVICE_ACCOUNT_NAME}" \
  --service-account-issuer-url "${SERVICE_ACCOUNT_ISSUER}"
```


## Create a registry credential in AKV for TAP install

In this install we are not using relocated images. This example uses direct access to tanzu net.

```bash
export REGISTRY_USER='user'
export REGISTRY_PASS='[password]'
export REGISTRY_HOST='registry.tanzu.vmware.com'
export REGISTRY_CONFIG=$(ytt -f cli/templates/dockerconfig.yml --data-value="registry.username=$REGISTRY_USER" --data-value="registry.hostname=$REGISTRY_HOST" --data-value="registry.password=$REGISTRY_PASS" --output=json | jq -c -r .dockerconfigjson | base64)

az keyvault secret set --vault-name $KEYVAULT_NAME --name "tap-registry-creds" --value $REGISTRY_CONFIG


```


## Create any senstive values files needed and put them in AKV

in this example there is only one needed and that is for the view cluster. you can place any tap values files with senstive data in the `cli/sensitive-values` folder.

```bash
export TAP_VIEW_VALUES=$(cat cli/sensitive-values/sensitive-view-values.yml | base64)

az keyvault secret set --vault-name $KEYVAULT_NAME --name "tap-view-values" --value $TAP_VIEW_VALUES
```



## Enable gitops on the cluster groups for your clusters

