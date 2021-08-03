---
layout: post
title:  "Django — Using Azure blob storage to handle static & media assets — from scratch"
author: davidsantiago
categories: [ django, azure ]
image: assets/images/django.png
---

> This post was originally published on [Medium](https://medium.com/@DawlysD/django-using-azure-blob-storage-to-handle-static-media-assets-from-scratch-90cbbc7d56be).


In this tutorial, we will see how to use the [Azure Blob service](https://azure.microsoft.com/en-us/services/storage/blobs/) to handle **static assets** & **media assets** (user uploaded files) with **Django**.

### Fundations 

I need to create a Resource Group, Storage Account and two containers. I will use the following naming convention:
*   Resource group name: _DemoDjangoBlob_
*   Storage Account name: _djangoaccountstorage_
*   Media container name: _media_
*   Static container name: _static_

## Resources creation using Azure CLI


```bash
# Create resource group named DemoDjangoBlob
$ az group create -l westeurope -n DemoDjangoBlob
# Create storage account named djangoaccountstorage
$ az storage account create -n djangoaccountstorage -g DemoDjangoBlob -l westeurope --sku Standard_LRS --https-only true
# (optional) Update storage account to V2
$ az storage account update -n djangoaccountstorage -g DemoDjangoBlob --set kind=StorageV2
# Create the "media" container_  
$ az storage container create -n media --account-name djangoaccountstorage --public-access blob
# Create the "static" container 
$ az storage container create -n static --account-name djangoaccountstorage --public-access blob
```

## Resources creation using Azure Portal

**Select** Subscription and **display** Resource group list. **Click** on “_Create Resource group_”.

![rg-creation]({{ site.baseurl }}/assets/images/django-blob-1.png)

**Click** on “_Create_”, **go** to the resource group and **Click** on “_Create resources_”.

**Search** “_storage account_” and **Click** on “_Create_”. **Configure** the Storage Account as following:

![sa-creation]({{ site.baseurl }}/assets/images/django-blob-2.png)

**Enable** “_Secure transfer required_” in Advanced tab and **Create** the resource.

**Go** to the resource on “_Blob service_” item menu. **Create** `static` and `media` containers with “_Blob_” public access level :

![containers-access-level]({{ site.baseurl }}/assets/images/django-blob-3.png)

It is done. We will need to manage Storage Account Keys later, they are located in “_Access Keys_” item menu.

## Install and configure Django

The easiest way is to **install** everything using `pipenv` :

```bash
# Basic Django installation_  
$ mkdir demo-django-azure_blob  
$ cd demo-django-azure_blob/  
$ pipenv install django  
$ pipenv install django-storages[azure]  
$ pipenv shell  
(demo-django-azure_blob) $ django-admin.py startproject backend .  
(demo-django-azure_blob) $ python manage.py migrate
```

**Add** `storages` to `INSTALLED_APPS` in _settings.py_ module:

```python
INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    'storages',
]
```

Then, **Add** the following lines at the end of _settings.py_ module :

```python
DEFAULT_FILE_STORAGE = 'backend.custom_azure.AzureMediaStorage'
STATICFILES_STORAGE = 'backend.custom_azure.AzureStaticStorage'

STATIC_LOCATION = "static"
MEDIA_LOCATION = "media"

AZURE_ACCOUNT_NAME = "djangoaccountstorage"
AZURE_CUSTOM_DOMAIN = f'{AZURE_ACCOUNT_NAME}.blob.core.windows.net'
STATIC_URL = f'https://{AZURE_CUSTOM_DOMAIN}/{STATIC_LOCATION}/'
MEDIA_URL = f'https://{AZURE_CUSTOM_DOMAIN}/{MEDIA_LOCATION}/'
```

`AZURE_ACCOUNT_NAME` **must be replaced** by the storage account name.

Finally, **create** a _custom_azure.py_ file in _backend/_ folder and  **replace** `account_name` and `account_key` values:

```python
from storages.backends.azure_storage import AzureStorage

class AzureMediaStorage(AzureStorage):
    account_name = 'djangoaccountstorage' # <storage_account_name>
    account_key = 'your_key_here' # <storage_account_key>
    azure_container = 'media'
    expiration_secs = None

class AzureStaticStorage(AzureStorage):
    account_name = 'djangoaccountstorage' # <storage_account_name>
    account_key = 'your_key_here' # <storage_account_key>
    azure_container = 'static'
    expiration_secs = None
```

**Get storage account key** with Azure CLI :

```bash
$ az storage account keys list -n djangoaccountstorage -g DemoDjangoBlob
```

It is almost finished. Azure resources and Django are ready, let's migrate static & media files now.

## Migrate static files

`collectstatic` command will copy files to remote location now.

```bash
(demo-django-blob) $ python manage.py collectstatic
You have requested to collect static files at the destination
location as specified in your settings.
This will overwrite existing files!
Are you sure you want to do this?
Type 'yes' to continue, or 'no' to cancel: yes
119 static files copied
```

119 static files were copied but I have an “empty” project. In fact, Django Admin static files (css, fonts, js and img) were copied.

## Test it

**Run the local server** without static files:

```bash
(demo-django-blob) $ python manage.py runserver --nostatic
```

**Navigate to** [http://127.0.0.1:8000/admin/login/?next=/admin/](http://127.0.0.1:8000/admin/login/?next=/admin/)

We have the following Django administration page :

![django-admin]({{ site.baseurl }}/assets/images/django-blob-4.png)

**Check** source code:

![page-source]({{ site.baseurl }}/assets/images/django-blob-5.png)

## Conclusions

Using Azure blob service with Django is pretty easy thanks to [django-storages](https://django-storages.readthedocs.io/en/latest/) library. 
Moreover, features like [Azure CDN](https://github.com/jschneier/django-storages/pull/611) will come soon 😃.

Next steps can be :

*   Store `account_name` and `account_key` inside environment variables.
*   Deal with Cross-Origin Ressource Sharing supported by Azure Storage Services

Hope this tutorial was helpful. Feel free to leave me a feedback 😏.


PS: This tutorial was largely inspired by the tutorial from Vitor Freitas : [How to Setup Amazon S3 in a Django Project](https://simpleisbetterthancomplex.com/tutorial/2017/08/01/how-to-setup-amazon-s3-in-a-django-project.html).
