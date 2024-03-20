---
author: "Chendrayan Venkatesan"
title: "Unlocking Secrets: A Guide to Integrate Azure Key Vault with GitLab"
date: 2024-03-20
description: "If you want to keep a secret...keep it in Azure Key Vault"
tags: ["Azure", "Azure Federated App Credential", "OIDC" , "GitLab" , "GitLab Azure Key Vault"]
thumbnail: /2024/03/Security-1.jpg
---

# Introduction

Lately, I have experienced issues using other popular key vault providers. So, I demonstrated using the Azure key vault in Azure DevOps. A question popped up: How do we use Az key vault in GitLab? 

Below links gives the direction...

- [GitLab & Azure Key Vault](https://docs.gitlab.com/ee/ci/secrets/azure_key_vault.html)
- [Azure AD Federated Identity Credential](https://docs.gitlab.com/ee/ci/cloud_services/azure/index.html#create-azure-ad-federated-identity-credentials)

Isn’t it like breathing? Nope! It is not! This blog post is to show you the easiest way! 

> I followed the instructions and came up with the most straightforward way 

# Video 

I respect your time and recorded the demonstration. 

{{<youtube xiF4DkYjdzA>}}

## Prerequisites 

- Azure Subscription with appropriate permission
- Azure CLI (Optional) 
- Azure PowerShell 
- GitLab (I am using the free tier for demonstration)

# Solution

```PowerShell
#Create New Service Principal 
New-AzADServicePrincipal -DisplayName '<STRING>'

#Assign reader permision for the SPN on the Azure subscription
New-AzRoleAssignment -ObjectId '<SPN Object ID>' -Scope '/subscriptions/<SUBSCRIPTION ID>' -RoleDefinitionName 'Reader'

#Grant permission on the Azure key vault (For demo, I am granting admin permission)
New-AzRoleAssignment -ApplicationId '<APPLICATION ID>' -Scope '/subscriptions/<SUBSCRIPTION ID>/resourceGroups/<RESOURCE GROUP NAME>/providers/Microsoft.KeyVault/vaults/<KEY VAULT NAME>' -RoleDefinitionName 'Key Vault Administrator'

#Create Azure AD application federated credential
$Params = @{
    ApplicationObjectId = 'APPLICATION OBJECT ID'
    Audience            = 'https://gitlab.com' 
    Issuer              = 'https://gitlab.com' 
    Name                = 'gitlab-federated-identity' 
    Subject             = 'project_path:<GROUP>/<PROJECT>:ref_type:branch:ref:<BRANCH NAME>' 
    Description         = 'GL Service Account Federated Identity' 
}

New-AzADAppFederatedCredential @Params
```

## Explanation

- The `New-AzADServicePrincipal` cmdlet creates a new service principal with the display name.
- The `New-AzRoleAssignment` cmdlet assigns the ***Reader*** role to the service principal on the specified Azure subscription.
- Another `New-AzRoleAssignment` cmdlet grants admin permission on the `Azure Key Vault`.
- Finally, the script creates an Azure AD application federated credential using the specified parameters.

# References

- [Overview of federated identity credentials in Microsoft Entra ID](https://learn.microsoft.com/en-us/graph/api/resources/federatedidentitycredentials-overview?view=graph-rest-1.0)
- [Creating Federated Identity Credential](https://learn.microsoft.com/en-us/graph/api/application-post-federatedidentitycredentials?view=graph-rest-1.0&tabs=http)