---
layout: post
title:  "NSG Flow Logs to Event Hubs using Logstash & Container Apps"
author: davidsantiago
categories: [ azure, network, monitoring ]
image: assets/images/nsg-flow-logs-to-event-hubs.png
featured: true
---

[NSG flow logs](https://learn.microsoft.com/en-us/azure/network-watcher/network-watcher-nsg-flow-logging-overview) is a feature of Azure Network Watcher that allows you to log information about IP traffic flowing through a network security group (NSG). 

The current **limitation** is that the flow data can **only be sent to Azure Storage**. Sending this flow data to Azure Event Hubs, which is the preferred destination for most Azure logs and metrics these days, is not possible.

This article demonstrates the process of utilizing Logstash within Azure Container Apps to send NSG flow logs from Azure Storage Account to Azure Event Hubs.

# Logstash Docker image

[Logstash](https://www.elastic.co/logstash/) is a tool for managing events and logs. It has an official [Docker image](https://hub.docker.com/_/logstash).

Let's enhance this image by incorporating two Logstash modules:
1. [logstash-input-azure_blob_storage](https://github.com/janmg/logstash-input-azure_blob_storage)
2. [logstash-output-azure_event_hubs](https://github.com/bryanklewis/logstash-output-azure_event_hubs)


* Create a `Dockerfile`:

```Dockerfile
FROM docker.elastic.co/logstash/logstash:8.7.1

RUN logstash-plugin install logstash-input-azure_blob_storage
RUN logstash-plugin install logstash-output-azure_event_hubs

ENV XPACK_MONITORING_ENABLED=false
```

* Build the image:

```bash
docker build -t logstash_blob_eventhubs:1 .
```

* Create an Azure Container Registry and push `logstash_blob_eventhubs:1` image:

```bash
RESOURCE_GROUP="demo-logstash"
ACR_NAME="myacrnsgdemo"
LOCATION="westeurope"

# RG creation
az group create --location $LOCATION --name $RESOURCE_GROUP

# ACR creation
az acr create --resource-group $RESOURCE_GROUP --name $ACR_NAME --sku Basic --location $LOCATION

# ACR login
az acr login --name $ACR_NAME

# Docker tag
docker tag logstash_blob_eventhubs:1 $ACR_NAME.azurecr.io/logstash_blob_eventhubs:1

# Push Docker image to ACR
docker push $ACR_NAME.azurecr.io/logstash_blob_eventhubs:1
```

# Logstash configuration file

Let's create the Logstash configuration file and store it into an Azure Files volume to be used by Azure Container apps:

* Create `logstash-nsg.conf` file:

```vim

input {
  azure_blob_storage {
    storageaccount => "${INPUT_SA_NAME}" 
    access_key => "${INPUT_SA_KEY}" 
    container => "insights-logs-networksecuritygroupflowevent"
    codec => "json"
    logtype => "nsgflowlog"
    prefix => "resourceId=/"
    path_filters => ['**/*.json']
    addfilename => true 
    addall => true
    interval => 60 
    debug_timer => true 
    debug_until => 1000 
    addall => true
    registry_create_policy => "resume"
  }
}

output {
  stdout {
    codec => json
  }
  azure_event_hubs {
    service_namespace => "${OUTPUT_EH_NS}" # Exclude .servicebus.windows.net
    event_hub => "${OUTPUT_EH}" 
    sas_key_name => "${OUTPUT_EH_SAS_KEY_NAME}" 
    sas_key => "${OUTPUT_EH_SAS_KEY}"
  }
}
```

> `stdout` output is useful only for troubleshooting purpose. Remove this block for production usage.

* Create the Storage Account:

```bash

SA_NAME="acacfgsa"

az storage account create \
  --name $SA_NAME \
  --resource-group $RESOURCE_GROUP \
  --location $LOCATION \
  --sku Standard_LRS \
  --kind StorageV2
```

* Create File Share in the Storage Account to store Logstash configuration:

```bash
az storage share-rm create -g $RESOURCE_GROUP --storage-account $SA_NAME --name logstashconfiguration --quota 1
```

* Upload `logstash-nsg.conf` to `logstashconfiguration` File Share

```bash
az storage file upload --account-name $SA_NAME --path logstash-nsg.conf --share-name logstashconfiguration --source logstash-nsg.conf
```

# Azure Container Apps

Let's now create a Container App Environment, a Container App using previously pushed Docker image and Logstash configuration file in Azure Files:

* Create Container Apps Environment:

```bash
ACA_ENV_NAME="aca-env"

az containerapp env create --name $ACA_ENV_NAME --resource-group $RESOURCE_GROUP --location $LOCATION
```

* Configure Container Apps Environment to use Azure Files Share:

```bash

AZURE_ACCOUNT_KEY=$(az storage account keys list \
    --account-name $SA_NAME \
    --resource-group $RESOURCE_GROUP \
    --query "[0].value" \
    -o tsv)

az containerapp env storage set --name $ACA_ENV_NAME --resource-group $RESOURCE_GROUP \
    --storage-name $SA_NAME \
    --azure-file-account-name $SA_NAME \
    --azure-file-account-key $AZURE_ACCOUNT_KEY \
    --azure-file-share-name logstashconfiguration \
    --access-mode ReadWrite
```

* Create the Container app

```bash
ACA_APP_NAME="logstash"
ACR_USERNAME=$(az acr credential show -n $ACR_NAME --query 'username' -o tsv)
ACR_PASSWORD=$(az acr credential show -n $ACR_NAME --query 'passwords[0].value' -o tsv)

# Input - Storage Account containing NSG Flow Logs
NSG_FLOW_LOGS_STORAGE_ACCOUNT_NAME="<VALUE>"
NSG_FLOW_LOGS_STORAGE_ACCOUNT_KEY="<VALUE>"

# Output - Event Hubs
EVENT_HUBS_NAMESPACE="<VALUE>"
EVENT_HUBS_ENTITY="<VALUE>"
EVENT_HUBS_KEY_NAME="<VALUE>"
EVENT_HUBS_KEY="<VALUE>"

az containerapp create -n $ACA_APP_NAME -g $RESOURCE_GROUP \
    --image $ACR_NAME.azurecr.io/logstash_blob_eventhubs:1 \
    --environment $ACA_ENV_NAME \
    --registry-server $ACR_NAME.azurecr.io \
    --registry-username $ACR_USERNAME \
    --registry-password $ACR_PASSWORD \
    --secrets input-sa-name="$NSG_FLOW_LOGS_STORAGE_ACCOUNT_NAME" input-sa-key="$NSG_FLOW_LOGS_STORAGE_ACCOUNT_KEY" output-eh-ns="$EVENT_HUBS_NAMESPACE" output-eh="$EVENT_HUBS_ENTITY" output-eh-sas-key-name="$EVENT_HUBS_KEY_NAME" output-eh-sas-key="$EVENT_HUBS_KEY" \
    --env-vars "INPUT_SA_NAME=secretref:input-sa-name" "INPUT_SA_KEY=secretref:input-sa-key" "OUTPUT_EH_NS=secretref:output-eh-ns" "OUTPUT_EH=secretref:output-eh" "OUTPUT_EH_SAS_KEY_NAME=secretref:output-eh-sas-key-name" "OUTPUT_EH_SAS_KEY=secretref:output-eh-sas-key" \
    --cpu 2 \
    --memory 4.0Gi \
    --min-replicas 1 \
    --max-replicas 1
```

* Edit the Container App to configure volume:

```bash
az containerapp show -n $ACA_APP_NAME -g $RESOURCE_GROUP -o yaml > app.yaml

vi app.yaml
```

* Add bold lines: 

<pre><code>
  template:
    containers:
    - image: myacrnsgdemo.azurecr.io/logstash_blob_eventhubs:1
      name: logstash
      resources:
        cpu: 2
        ephemeralStorage: 2Gi
        memory: 4Gi
      <strong>volumeMounts:
      - volumeName: logstash-config
        mountPath: /usr/share/logstash/pipeline/
    volumes:
    - name: logstash-config
      storageType: AzureFile
      storageName: acacfgsa</strong>
</code></pre>

* Update the container app using the YAML file:

```bash
az containerapp update --name $ACA_APP_NAME --resource-group $RESOURCE_GROUP --yaml app.yaml
```

* Check container app log stream:

  ![container-app-log-stream]({{ site.baseurl }}/assets/images/nsg-logstash-logstream.png)

* Check container app metrics:
  
  ![container-app-metrics]({{ site.baseurl }}/assets/images/nsg-ca-metrics-nob.png)

* Check Event Hubs metrics

  ![event-hub-metrics]({{ site.baseurl }}/assets/images/nsg-eh-metrics.png)

# Conclusion

That's all! NSG Flow Logs are now sent to Event Hubs using Logstash running in Azure Container Apps.

Other outputs, like [http output](https://www.elastic.co/guide/en/logstash/current/plugins-outputs-http.html) can be used if needed.