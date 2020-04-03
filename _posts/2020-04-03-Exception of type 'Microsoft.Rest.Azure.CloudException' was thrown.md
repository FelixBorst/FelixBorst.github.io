---
title: "Exception of type 'Microsoft.Rest.Azure.CloudException' was thrown."
date: 2020-04-03 14:22
categories: [Azure, PowerShell, KeyVault, Access Policies]
tags: [azure, powershell, automation, keyvault, error, fix, access policy]
toc: false
---

### Introduction ###

I recently came across an interesting error during pipeline execution of PowerShell scripts using Azure DevOps Release pipelines.

Before I dive into detail, here is the not-so-self-explanatory error message:

```
Error: Exception of type 'Microsoft.Rest.Azure.CloudException' was thrown.
```

So I started off my investigation. First point was to sign in as the service principal who was actually executing the script, as I was suspecting an RBAC-issue from my previous experience with Azure KeyVault.

I soon realized the issue was really RBAC-related. But it was not a permission issue on an Azure resource but an access-issue on the Azure AD instead.

### So why does this issue appear? ###

It appears, because Azure does a validation check on object IDs which are related to service principals or user objects. It basically checks if the user really exists before trying to assign him a certain permission. But if the user executing the command does not have access to the Azure AD at all, then this script fails because a validation is not possible.

### The fix ###

When running ``` Set-AzKeyVaultAccessPolicy ```

make sure to include the flag ```-BypassObjectIdValidation``` to make the command work again. Kudos to the user 'ratnakarsinha' on GitHub for posting the issue here:  
https://github.com/Azure/azure-powershell/issues/10029

Unfortunately ``` Set-AzKeyVaultAccessPolicy ``` is not the only command which is suffering from this permission issue. Commands like ``` Get-AzRoleAssignment ``` have a similar issue but do not offer the same flag as ``` Set-AzKeyVaultAccessPolicy ``` does. To solve this issue in these cases make sure your user running the command is whitelisted in the **Microsoft-Graph-API** to read directory data.