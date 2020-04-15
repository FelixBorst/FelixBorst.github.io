---
title: "Deploy an Azure AD B2C Instance using PowerShell and ARM Templates."
date: 2020-04-15 10:48
categories: [Azure, Azure AD, PowerShell, B2C, Tenant, ARM]
tags: [azure, powershell, automation, b2c, tenant, ad, aad, azure ad, arm, templates]
toc: false
---

## Introduction ##

During a project we recently had the question pop up: "What if we want to automate the deployment of an Azure AD B2C instance?".  
My answer: "Let's do it".

To be honest it is not as hard as you may think it is. There are a few catches here and there, but I'm gonna highlight them in this post.

## The concept ##

An Azure AD B2C Instance is similar to the good old B2B instance most of you should be familiar with.  
The key difference lies in it's name: It's customer focused. Meaning it offers a different feature set compared to the B2B variant.  
For a full comparison check the [documentation](https://docs.microsoft.com/en-us/azure/active-directory/b2b/compare-with-b2c).  
But back to topic, how to you deploy it?  
Well, first of all you have to understand that this is a two steps process:

1. Creating the Azure AD B2C Instance.
2. Making it manageable through the Azure Portal (aka. attaching it to a resource group).

## The process ##

### Creating the Azure AD B2C Instance ###

The first steps requires an Azure AD API Call and that's why we need an access token.  
You may be familiar with the typical Azure Access Tokens, however this one is slightly different as we are not calling `management.azure.com` but `main.iam.ad.ext.azure.com` instead. As you need to generate an access token with the target URL in mind, we are going a slightly different route.  
Thanks to JustinGrote on GitHub, there is an easy way of retrieving the token from the current Azure context without much code-writing.  
If you want to check his code out, I've attached a link to the sample he posted: [JustinGrote on GitHub](https://github.com/JustinGrote/Az.PortalAPI/blob/master/Az.PortalAPI/Public/Get-Token.ps1).  

Here's the code we will use:

```powershell
$context = Get-AzContext
$resource = '74658136-14ec-4630-ad9b-26e160ff0fc6'
$token = [Microsoft.Azure.Commands.Common.Authentication.AzureSession]::Instance.AuthenticationFactory.Authenticate($context.Account, $context.Environment, $context.Tenant.Id, $null, "Never", $null, $resource)
```

`$resource` points to a GUID. This GUID references an built-in Azure AD Application which has the base URL `main.iam.ad.ext.azure.com`. So make sure your GUID matches up!  

This next section performs the actual API call. You can see that we inserted the token with the Bearer prefix. Also important: `$B2cCompanyName`, `$B2cCountryCode` and  `$B2cDomainPrefix` need to be valid values, e.g.:

- `$B2cCompanyName` is the name the B2C Tenant will get.
- `$B2cCountryCode` is the country code where the B2C tenant's data will be hosted. For more info check [here](https://docs.microsoft.com/en-us/azure/active-directory-b2c/data-residency)
- `$B2cDomainPrefix` is the prefix of the tenant's URL. The full url will look like this: `$B2cDomainPrefix.onmicrosoft.com`

The script returns the tenantId. We will save that one to the variable `$b2CTenantId`. Make sure you don't loose this one, otherwise you will not have a way to access this tenant later on.

```powershell
$result = Invoke-WebRequest `
    -Uri "https://main.iam.ad.ext.azure.com/api/Directories/B2C" `
    -Method "POST" `
    -Headers @{ 'Authorization' = "Bearer $($token.AccessToken)" } `
    -ContentType "application/json" `
    -Body "{`"companyName`":`"$B2cCompanyName`",`"countryCode`":`"$B2cCountryCode`",`"initialDomainPrefix`":`"$B2cDomainPrefix`"}"

$b2CTenantId = $result.Content.Replace('"', '')
```

### Attaching the B2C Tenant to an Azure Resource Group ###

For this action, we will need the following ARM-Template:

```json
{
    "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
    "contentVersion": "1.0.0.0",
    "parameters": {
        "Location": {
            "type": "String",
            "defaultValue": "Europe"
        },
        "TenantId": {
            "type": "String"
        },
        "Name": {
            "type": "String"
        }
    },
    "resources": [
        {
            "type": "Microsoft.AzureActiveDirectory/b2cDirectories",
            "apiVersion": "2017-01-30",
            "name": "[parameters('Name')]",
            "location": "[parameters('Location')]",
            "tags": {
            },
            "sku": {
                "name": "Standard",
                "tier": "A0"
            },
            "properties": {
                "tenantId": "[parameters('TenantId')]"
            }
        }
    ]
}
```

This ARM-Template needs three input parameters:

- `Location`: Depending on what you've used above for `$B2cCountryCode` you have to pick the correct location. The matching regions can be found [here](https://docs.microsoft.com/en-us/azure/active-directory-b2c/data-residency#data-residency).  
As an example: If `$B2cCountryCode` is `DE` (for Germany), the matching region would be `Europe`.
- `TenantId`: This is the tenant ID which gets returned by executing the API-Call above. If you've run the script it should be stored in the variable `$b2CTenantId`.
- `Name`: This is the full name of the tenant. Should be '`$B2cDomainPrefix`.onmicrosoft.com'.

The B2C-Instance should now appear in the resource group, into which you deployed the ARM-Template.  

Hope this helped you - if you have any questions, feel free to hit me up :)

