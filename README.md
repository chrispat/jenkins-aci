# Mounting volumes in container instance

https://docs.microsoft.com/en-us/azure/container-instances/container-instances-mounting-azure-files-volume
http://unethicalblogger.com/2017/07/31/jenkins-azure-container-instances.html

## Create all of the resources and deploy the container group with persistent home.

Currently this does not seem to work due to an error in setting the initialAdminPassword

```sh
# Change these four parameters
ACI_PERS_STORAGE_ACCOUNT_NAME=jenkinshome$RANDOM
ACI_PERS_RESOURCE_GROUP=$USER-jenkins-aci
ACI_PERS_LOCATION=eastus
ACI_PERS_SHARE_NAME=jenkinshome

az group create --name $ACI_PERS_RESOURCE_GROUP --location $ACI_PERS_LOCATION

# Create the storage account with the parameters
az storage account create -n $ACI_PERS_STORAGE_ACCOUNT_NAME -g $ACI_PERS_RESOURCE_GROUP -l $ACI_PERS_LOCATION --sku Standard_LRS

# Export the connection string as an environment variable, this is used when creating the Azure file share
export AZURE_STORAGE_CONNECTION_STRING=`az storage account show-connection-string -n $ACI_PERS_STORAGE_ACCOUNT_NAME -g $ACI_PERS_RESOURCE_GROUP -o tsv`

# Create the share
az storage share create -n $ACI_PERS_SHARE_NAME

# Get storage account
STORAGE_ACCOUNT=$(az storage account list --resource-group $ACI_PERS_RESOURCE_GROUP --query "[?contains(name,'$ACI_PERS_STORAGE_ACCOUNT_NAME')].[name]" -o tsv)
echo $STORAGE_ACCOUNT

# get storage key
STORAGE_KEY=$(az storage account keys list --resource-group $ACI_PERS_RESOURCE_GROUP --account-name $STORAGE_ACCOUNT --query "[0].value" -o tsv)
echo $STORAGE_KEY

# Setup keyvalut
KEYVAULT_NAME=jenkins-aci-keyvault
az keyvault create -n $KEYVAULT_NAME --enabled-for-template-deployment -g $ACI_PERS_RESOURCE_GROUP

# Add storage key to vault
KEYVAULT_SECRET_NAME=jenkinshomestoragekey
az keyvault secret set --vault-name $KEYVAULT_NAME --name $KEYVAULT_SECRET_NAME --value $STORAGE_KEY

# Get keyvalue resource id
echo $STORAGE_ACCOUNT
echo $KEYVAULT_SECRET_NAME
az keyvault show --name $KEYVAULT_NAME --query [id] -o tsv

# deploy the resource group
az group deployment create --name $USER-jenkins-aci-deployment --template-file jenkins-aci-template.json --parameters jenkins-aci-parameters.json --resource-group $ACI_PERS_RESOURCE_GROUP

az container logs --resource-group $ACI_PERS_RESOURCE_GROUP --name jenkins-ci --container-name jenkins-ci

```

## Create the resources and deploy the container group without persistent home

```sh
# Change these four parameters
ACI_PERS_RESOURCE_GROUP=$USER-jenkins-aci
ACI_PERS_LOCATION=eastus

# Create the resource group
az group create --name $ACI_PERS_RESOURCE_GROUP --location $ACI_PERS_LOCATION

# deploy the resource group
az group deployment create --name $USER-jenkins-aci-deployment --template-file jenkins-aci-template-no-storage.json --resource-group $ACI_PERS_RESOURCE_GROUP

# read the logs to get the initial admin password
az container logs --resource-group $ACI_PERS_RESOURCE_GROUP --name jenkins --container-name jenkins-ci

az container delete --resource-group $ACI_PERS_RESOURCE_GROUP --name jenkins-ci

```