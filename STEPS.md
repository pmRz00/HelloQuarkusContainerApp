PREFIX=quarkustostille
LOCATION=northeurope
RG_NAME=$PREFIX"rg"
ACR_NAME=$PREFIX"acr"
LOG_ANALYTICS_WORKSPACE=$PREFIX"law"
CONTAINERAPPS_ENVIRONMENT=$PREFIX"cae"

# log in to Azure
az login --use-device-code

# add container apps CLI extension (Azure Container App is still in preview!)
az upgrade
az extension add \
  --source https://workerappscliextension.blob.core.windows.net/azure-cli-extension/containerapp-0.2.0-py2.py3-none-any.whl
az provider register --namespace Microsoft.Web

# create a resource group
az group create -n $RG_NAME -l $LOCATION

# create an ACR
az acr create -n $ACR_NAME -g $RG_NAME -l $LOCATION --sku Basic --admin-enabled true

# log in to ACR
az acr login -n $ACR_NAME 

# get ACR login server
ACR_LOGIN_SERVER=`az acr show -n $ACR_NAME --query loginServer --out tsv`

# get ACR password 
ACR_PASSWORD=`az acr credential show -n $ACR_NAME --query passwords[0].value  --out tsv`

# scaffold a new Quarkus project 
mvn io.quarkus.platform:quarkus-maven-plugin:2.5.4.Final:create \
    -DprojectGroupId=org.acme \
    -DprojectArtifactId=getting-started \
    -DclassName="org.acme.getting.started.GreetingResource" \
    -Dpath="/hello"

# cd into new project folder
cd getting-started

# package native (no JVM) app
mvn package -Pnative

# build image
docker build -f src/main/docker/Dockerfile.native -t quarkus/getting-started .

# tag image
docker tag quarkus/getting-started $ACR_LOGIN_SERVER/quarkus-quickstart/getting-started

# push image
docker push $ACR_LOGIN_SERVER/quarkus-quickstart/getting-started

IMAGE_NAME=$ACR_LOGIN_SERVER/quarkus-quickstart/getting-started:latest

# manually verfify image exists
az acr repository list -n $ACR_NAME

# set up az monitor and log analytics workspace
az monitor log-analytics workspace create \
    --resource-group $RG_NAME \
    --workspace-name $LOG_ANALYTICS_WORKSPACE

LOG_ANALYTICS_WORKSPACE_CLIENT_ID=`az monitor log-analytics workspace show --query customerId -g $RG_NAME -n $LOG_ANALYTICS_WORKSPACE --out tsv`
LOG_ANALYTICS_WORKSPACE_CLIENT_SECRET=`az monitor log-analytics workspace get-shared-keys --query primarySharedKey -g $RG_NAME -n $LOG_ANALYTICS_WORKSPACE --out tsv`

# create container apps environment
az containerapp env create \
    --name $CONTAINERAPPS_ENVIRONMENT \
    --resource-group $RG_NAME \
    --logs-workspace-id $LOG_ANALYTICS_WORKSPACE_CLIENT_ID \
    --logs-workspace-key $LOG_ANALYTICS_WORKSPACE_CLIENT_SECRET \
    --location $LOCATION

# create container app
az containerapp create \
    --name hello \
    --resource-group $RG_NAME \
    --environment $CONTAINERAPPS_ENVIRONMENT \
    --image $IMAGE_NAME \
    --target-port 8080 \
    --ingress 'external' \
    --registry-login-server $ACR_LOGIN_SERVER \
    --registry-username $ACR_NAME \
    --registry-password $ACR_PASSWORD

echo `az containerapp show -n hello -g $RG_NAME --query configuration.ingress.fqdn --out tsv`/hello 