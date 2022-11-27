---
title: "GITLAB CICD TO DEPLOY AZURE FUNCTIONS"
date: 2022-11-27T20:32:21Z

draft: false
tags: ["GitLab", "KICS", "Azure" , "Azure Functions"]
type: post
showTableOfContents: true
---

![img](/images/avatar.png)

## Introduction

There are many ways to deploy Azure resources. 
- Click through the Azure portal to provision the resources. 
- Imperative
  - Use Az CLI, PowerShell, REST API etc. 
- Declarative
  - IaC – Bicep,Terraform, Pulumi etc. 

The most commonly used method to provision Azure resources is with Infrastructure as Code, and the tool is your choice. I recommend using Bicep. But, for this blog post, I used Terraform because I have to demonstrate the remote state backend configuration. 

## Design (Basic Solution)

![img](/images/blog/Blog-Pic-2.png)

## Resource Visualizer

> PRD - Production Environment

![img](/images/blog/Blog-Pic-3.png)

## Prerequisites

- Mandatory
    - Azure Account
    - GitLab Account
    - GitLab **P**ersonal **A**ccess **T**oken (PAT)
- Optional (required for testing in local dev machine)
    - Terraform CLI (Option)
    - Azure Functions Core CLI

---
:flashlight: **Azure Function App** is an instance to run the **Azure Functions.**

---

## Requirement

- Deploy Azure Function App.
- Deploy Azure Functions.
- Deploy solutions in multiple environments. 
- Implement KICS for Terraform.
- Terraform remote state management (GitLab). 
- Post the demo destroy the infrastructure. (lower environments).

## Solution

Our source code folder structure is as shown below

📦icollabrains  
 ┣ 📂src  
 ┃ ┣ 📂iHome  
 ┃ ┃ ┣ 📜function.json  
 ┃ ┃ ┗ 📜run.ps1  
 ┃ ┣ 📜.gitignore  
 ┃ ┣ 📜host.json  
 ┃ ┣ 📜local.settings.json  
 ┃ ┣ 📜profile.ps1  
 ┃ ┗ 📜requirements.psd1  
 ┣ 📂terraform  
 ┃ ┣ 📜backend.tf  
 ┃ ┣ 📜main.tf  
 ┃ ┣ 📜outputs.tf  
 ┃ ┣ 📜providers.tf  
 ┃ ┗ 📜variables.tf  
 ┣ 📜.gitlab-ci.yml  
 ┗ 📜README.md  

 ![images](/images/blog/Blog-Pic-1.png)

 ### Terraform Backend Configuration

> backend.tf

 ```Terraform 
terraform {
  backend "http" {
    address  = "${TF_ADDRESS}"
    username = "${TF_USERNAME}"
    password = "${TF_PASSWORD}"
  }
}
 ```

### Terraform Template

> main.tf

```Terraform
resource "azurerm_resource_group" "resource_group" {
  name     = var.resource_group_name
  location = var.location
}


resource "azurerm_storage_account" "storage_account" {
  name                     = var.storage_account_name
  resource_group_name      = azurerm_resource_group.resource_group.name
  location                 = azurerm_resource_group.resource_group.location
  account_tier             = var.storage_account_tier
  account_replication_type = var.storage_account_replication_type
}

resource "azurerm_app_service_plan" "app_service_plan" {
  name                = var.app_service_plan
  location            = azurerm_resource_group.resource_group.location
  resource_group_name = azurerm_resource_group.resource_group.name
  sku {
    tier = var.app_service_plan_tier
    size = var.app_service_plan_size
  }
}

resource "azurerm_windows_function_app" "function_app" {
  name                       = var.function_app_name
  resource_group_name        = azurerm_resource_group.resource_group.name
  location                   = azurerm_resource_group.resource_group.location
  storage_account_name       = azurerm_storage_account.storage_account.name
  storage_account_access_key = azurerm_storage_account.storage_account.primary_access_key
  service_plan_id            = azurerm_app_service_plan.app_service_plan.id
  site_config {
    application_stack {
      powershell_core_version = "7.2"
    }
    use_32_bit_worker = false
    always_on         = true
  }
}
```

### Variables

> variables.tf 

```Terraform
variable "resource_group_name" {
  type        = string
  description = "(optional) describe your variable."
}


variable "location" {
  type        = string
  description = "(optional) describe your variable."
}

variable "storage_account_name" {
  type        = string
  description = "(optional) describe your variable."
}

variable "storage_account_tier" {
  type        = string
  description = "(optional) describe your variable"
}

variable "storage_account_replication_type" {
  type        = string
  description = "(optional) describe your variable"
}

variable "app_service_plan" {
  type        = string
  description = "(optional) describe your variable"
}

variable "app_service_plan_tier" {
  type        = string
  description = "(optional) describe your variable"
}

variable "app_service_plan_size" {
  type        = string
  description = "(optional) describe your variable."
}

variable "function_app_name" {
  type        = string
  description = "(optional) describe your variable."
}
```

### References

- [Azure Functions](https://azure.microsoft.com/en-gb/products/functions/#overview)
- [Azure Functions Core Tools](https://github.com/Azure/azure-functions-core-tools)
- [Terraform](https://www.terraform.io/)

### Summary

Congratulations, you have successfully deployed the serverless solution. Now, you know to create a CICD pipeline in GitLab to deploy and destroy Azure functions. In my next blog post, I have plans to cover the below

- Implement Azure Traffic Manager for high availability
- Blue Green Deployment

---
:lightning: Your feedback is highly appreciated. Feel free to send your comments to my :envelope: [inbox](mailto:chendrayan.exchange@hotmail.com)

---
