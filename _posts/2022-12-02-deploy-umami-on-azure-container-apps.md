---
layout: post
title:  "Deploy umami on Azure Container Apps"
author: davidsantiago
categories: [ containers ]
image: assets/images/umami.png

---

[umami](https://umami.is/) is an [open source](https://github.com/umami-software/umami), privacy-focused alternative to Google Analytics.
This repo explains how to deploy it in Azure using [Azure Container Apps](https://learn.microsoft.com/en-us/azure/container-apps/overview) which is a new service that enables you to **run microservices and containerized applications on a serverless platform**.


# Prerequisites

* An Azure subscription 
* [Azure CLI](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli)
* A GitHub account with a [personal access tokens](https://github.com/settings/tokens/new) having at least `read:packages` right
    * This token will be used by Container Apps to pull docker image from GitHub registry.

# Deploy umami

## Define variables

**Update** `TO_CHANGE` string by appropriated value in below script and declare variables in your shell.

```
#############
# VARIABLES #
#############

# General
SUBSCRIPTION_ID="TO_CHANGE"
RESOURCE_GROUP="umami-rg"
LOCATION="westeurope"

# PostgreSQL
POSTGRESQL_NAME="TO_CHANGE" 
POSTGRESQL_ADMINUSER="TO_CHANGE"
POSTGRESQL_ADMINPASSWORD="TO_CHANGE"
POSTGRESQL_SKUNAME="Standard_B1ms"
POSTGRESQL_TIER="Burstable"
POSTGRESQL_VERSION="14"
POSTGRESQL_STORAGESIZE="32"
POSTGRESQL_DBNAME="umami"

# umami
UMAMI_REGISTRY_SERVER="ghcr.io"
UMAMI_IMAGE="ghcr.io/umami-software/umami:postgresql-v1.38.0" # Don't use images > 1.38.0 right now: https://github.com/umami-software/umami/issues/1604
UMAMI_REGISTRY_USERNAME="TO_CHANGE" # GitHub handle here
UMAMI_REGISTRY_PASSWORD="TO_CHANGE" # GitHub token with only "read:packages" right. Generate it from https://github.com/settings/tokens/new
UMAMI_HASHSALT="TO_CHANGE" # Example of salt: $(openssl rand -hex 32)
UMAMI_DATABASE_URL="postgres://$POSTGRESQL_ADMINUSER:$POSTGRESQL_ADMINPASSWORD@$POSTGRESQL_NAME.postgres.database.azure.com:5432/$POSTGRESQL_DBNAME?sslmode=require"

# ContainerApps
CONTAINERAPPS_ENVIRONMENT_NAME="umami-containerapp-environment"
CONTAINERAPPS_NAME="umami-containerapp"
```

## Configure CLI

```
az extension add --name containerapp --upgrade
az provider register --namespace Microsoft.App
az provider register --namespace Microsoft.OperationalInsights
```

## Create Azure Resource group

```
az account set -s "$SUBSCRIPTION_ID"
az group create --name "$RESOURCE_GROUP" --location "$LOCATION"
```

## Deploy Azure Database for PostgreSQL flexible server

```
##############
# POSTGRESQL #
##############

# Flexible service PostgreSQL creation
az postgres flexible-server create --name "$POSTGRESQL_NAME" --resource-group "$RESOURCE_GROUP"  \
  --location "$LOCATION" \
  --admin-user "$POSTGRESQL_ADMINUSER" \
  --admin-password "$POSTGRESQL_ADMINPASSWORD" \
  --sku-name "$POSTGRESQL_SKUNAME" \
  --tier "$POSTGRESQL_TIER" \
  --version "$POSTGRESQL_VERSION" \
  --storage-size "$POSTGRESQL_STORAGESIZE" \
  --public-access 0.0.0.0 --yes 

# Database creation
az postgres flexible-server db create --resource-group "$RESOURCE_GROUP" --server-name "$POSTGRESQL_NAME" --database-name "$POSTGRESQL_DBNAME"
```

## Deploy Container Apps with umami Docker image

```
##################
# CONTAINER APPS #
##################

az containerapp env create --name "$CONTAINERAPPS_ENVIRONMENT_NAME" --resource-group "$RESOURCE_GROUP" --location "$LOCATION"

az containerapp create \
  --name "$CONTAINERAPPS_NAME" \
  --resource-group "$RESOURCE_GROUP" \
  --environment "$CONTAINERAPPS_ENVIRONMENT_NAME" \
  --registry-server "$UMAMI_REGISTRY_SERVER" \
  --image "$UMAMI_IMAGE" \
  --registry-username "$UMAMI_REGISTRY_USERNAME" \
  --registry-password "$UMAMI_REGISTRY_PASSWORD" \
  --target-port 3000 \
  --ingress 'external' \
  --query properties.configuration.ingress.fqdn \
  --cpu 1.0 --memory 2Gi \
  --min-replicas 1 --max-replicas 1 \
  --env-vars HASH_SALT="$UMAMI_HASHSALT" DATABASE_TYPE=postgresql DATABASE_URL="$UMAMI_DATABASE_URL"
  ```
  
## Check it works

* Open URL given by the last command in your browser
* Login to umami with default credentials (login: admin ; password: umami)

  