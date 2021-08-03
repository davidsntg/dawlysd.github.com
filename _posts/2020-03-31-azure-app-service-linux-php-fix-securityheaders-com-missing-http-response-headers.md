---
layout: post
title:  "Azure App Service (Linux&PHP) — Fix (securityheaders.com) missing HTTP Response Headers"
author: davidsantiago
categories: [ azure ]
---

> This post was originally published on [Medium](https://medium.com/@DawlysD/azure-app-service-linux-php-fix-securityheaders-com-missing-http-response-headers-6389fe6f90d1).

This blog post explains how to fix headers anomalies given by [securityheaders.io](https://securityheaders.com/) on **Azure App Service** with **Linux & PHP** context.

## Step1 - Create the startup script for PHP

Unfortunately, we cannot use a web.config file when App Service is running Linux & Apache (normal: there is no IIS).

We need to create a shell script which will be executed when the container starts.

This shell script will modify the apache configuration for Headers service.

**Upload** bellow script to `/home/site/wwwroot/` folder:

```bash
#!/usr/bin/env bash
a2enmod headers

# Remove X-Powered-By
echo "Header always unset \"X-Powered-By\"" >> /etc/apache2/apache2.conf
echo "Header unset \"X-Powered-By\"" >> /etc/apache2/apache2.conf

# Strict-Transport-Security
echo "Header always set Strict-Transport-Security \"max-age=31536000; includeSubDomains\"" >> /etc/apache2/apache2.conf

# Content-Security-Policy
echo "Header set Content-Security-Policy \"default-src 'self' *.MYDOMAIN.com\"" >> /etc/apache2/apache2.conf

# Referrer-Policy
echo "Header set Referrer-Policy \"no-referrer-when-downgrade\"" >> /etc/apache2/apache2.conf

# X-Frame-Options
echo "Header set X-Frame-Options \"sameorigin\"" >> /etc/apache2/apache2.conf

# X-Frame-Options
echo "Header set X-Content-Type-Options \"nosniff\"" >> /etc/apache2/apache2.conf

# X-XSS-Protection	
echo "Header set X-XSS-Protection \"1; mode=block\"" >> /etc/apache2/apache2.conf

/usr/sbin/apache2ctl -D FOREGROUND
```

It can be done with :
* Browser: Azure Portal => App Service => Development Tools => SSH => `vi /home/site/wwwroot/startup.sh` && `chmod +x /home/site/wwwroot/startup.sh` 
* Traditional FTP Client

## Step2 - Configure App Service Startup Command

**Go** to Configuration tab => General settings and **update** startup command:
![app-service-configuration]({{ site.baseurl }}/assets/images/app-service-linux-php-1.png)

## Step3 - Restart

From Overview tab, Restart App Service and check logs using “App Service Logs” tab.

## Step4 - Test

Check the result:
* Using [securityheaders.io](https://securityheaders.com/)
* Old school: `curl --head https://myawesomesite.com`


### Source

This blog post is widely copied from the below articles:
* [How to fix the HTTP response headers on Azure Web Apps to get an A+ on securityheaders.io](https://tomssl.com/2016/06/30/how-to-fix-the-http-response-headers-on-azure-web-apps-to-get-an-a-plus-on-securityheaders-io/)
* [CUSTOM STARTUP SCRIPT FOR PHP — AZURE APP SERVICE LINUX](http://jsandersblog.azurewebsites.net/2019/08/06/custom-startup-script-for-php-azure-app-service-linux/)
