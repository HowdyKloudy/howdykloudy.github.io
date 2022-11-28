---
title: "GITLAB CICD TO DEPLOY AZURE FUNCTIONS"
date: 2022-11-27T20:32:21Z

draft: false
tags: ["GitLab", "KICS", "Azure" , "Azure Functions"]
type: post
showTableOfContents: true
---

![img](/images/blog/cover-01.jpg)

## Introduction

There are many ways to deploy Azure resources.   

- Azure Portal - Click and follow the UI!  
- Imperative  
  :rocket: Use Az CLI  
  :rocket: PowerShell  
  :rocket: REST API etc. 
- Declarative
  :rocket: IaC – Bicep,Terraform, Pulumi etc. 

The most commonly used method to provision Azure resources is with Infrastructure as Code, and the tool is your choice. I recommend using Bicep. But, for this blog post, I used Terraform because I have to demonstrate the remote state backend configuration. 

## Design (Initial Draft)

![img](/images/blog/Blog-Pic-2.png)

## Resource Visualizer

> PRD - Production Environment

![img](/images/blog/Blog-Pic-3.png)

## Prerequisites

- Mandatory
    - Azure Account
    - GitLab Account
    - GitLab **P**ersonal **A**ccess **T**oken (PAT)
      - [More about PAT](https://docs.gitlab.com/ee/user/profile/personal_access_tokens.html)
- Optional (required for testing in local dev machine)
    - Terraform CLI
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
- Pipeline Configuration & Rules
  - An implementor should pass a value for the environment and spin up the infrastructure.
  - On a merge request event, the below jobs should run
    - kics-scan
    - kics-result
    - validate
  - On a merge, the below jobs should run
    - plan
    - apply
    - deploy
    - functionaltest
    - Destory (manual)
## Solution

> Our source code folder structure is as shown below

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


A solution is not only to solve the problem. It should be easy to adapt! With that consideration, I would like to walk through the proof of concept deployment in simple steps. 

### Terraform Backend Configuration

Gitlab provides a Terraform HTTP backend to store the state files with the below essential features

- Version your Terraform state files.
- Encrypt the state file both in transit and at rest.
- Lock and unlock states.
- Remotely execute the terraform plan & apply commands.


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

| Variable    | Value                                                                              |
| ----------- | ---------------------------------------------------------------------------------- |
| TF_ADDRESS  | https://gitlab.com/api/v4/projects/[PROJECT_ID]/terraform/state/[ENVIRONMENT] |
| TF_USERNAME | GITLAB_USER_NAME                                                                   |
| TF_PASSWORD | GITLAB_PAT_TOKEN                                                                   |

:rocket: PROJECT_ID = ${CI_PROJECT_ID} (GitLab Predefined Variables)  
:rocket: ENVIRONMENT = ${ENVIRONMENT} (Var in YAML)

### Terraform Template

List of resources to provision

-	Resource Group
-	Storage Account
-	App Service Plan
-	Function App

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

> Variable Values

| Variable                         | Value                                   |
| -------------------------------- | --------------------------------------- |
| resource_group_name              | rgp-${Function_App_Name}-${Environment} |
| location                         | ukwest (CICD Variables - GitLab)        |
| storage_account_name             | ${Function_App_Name}stg${Environment}   |
| storage_account_tier             | Standard (CICD Variables - GitLab)      |
| storage_account_replication_type | LRS (CICD Variables - GitLab)           |
| app_service_plan                 | ${Function_App_Name}-${Environment}     |
| app_service_plan_tier            | Standard (CICD Variables - GitLab)      |
| app_service_plan_size            | S1 (CICD Variables - GitLab)            |
| function_app_name                | ${Function_App_Name}-${Environment}     |

The below values are declared in the YAML file. 

- Function_App_Name = icollabrains
- Environment = dev | prd | stage

### CICD

#### Variables 
```YAML
variables:
  Environment: dev
  Function_App_Name: "icollabrains"
  TF_ADDRESS: "https://gitlab.com/api/v4/projects/${CI_PROJECT_ID}/terraform/state/${Environment}"
  KICS_VERSION: "1.2.4"
```
- Environment - Value can be of anything like dev, uat, test or prod. 
- Function_App_Name - Name of your function app as applicable 
- TF_ADDRESS - HTTP address for storing the state file
- KICS_VERSION - KICS binary file version number
#### Before Script Template

```YAML
.before_script_template:
  image:
    name: hashicorp/terraform:latest
    entrypoint:
      - "/usr/bin/env"
      - "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
  before_script:
    - rm -rf .terraform
    - terraform init -backend-config=address=${TF_ADDRESS} -backend-config=lock_address=${TF_ADDRESS}/lock -backend-config=unlock_address=${TF_ADDRESS}/lock -backend-config=username=${TF_USERNAME} -backend-config=password=${TF_PASSWORD} -backend-config=lock_method=POST -backend-config=unlock_method=DELETE -backend-config=retry_wait_min=5
```
> Terraform initialized with the below parameters

- backend-config=address=${TF_ADDRESS} 
- backend-config=lock_address=${TF_ADDRESS}/lock 
- backend-config=unlock_address=${TF_ADDRESS}/lock 
- backend-config=username=${TF_USERNAME} 
- backend-config=password=${TF_PASSWORD} 
- backend-config=lock_method=POST 
- backend-config=unlock_method=DELETE 
- backend-config=retry_wait_min=5

Lots of variables, right? Yeah, but not complex! These vars are to define the backend configuration with address, lock address, credentials, lock method, unlock method and retry wait time.

--- 
#### Stages

```YAML
stages:
  - kics-scan
  - kics-result
  - validate
  - plan
  - apply
  - deploy
  - functionaltest
  - destroy
```

For convenience, I defined eight simple stages (KICS scan, process the Scan result, )

---
#### KICS Scan

```YAML
kics-scan:
  stage: kics-scan
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
  image: alpine
  before_script:
    - apk add --no-cache libc6-compat
    - wget -q -c "https://github.com/Checkmarx/kics/releases/download/v${KICS_VERSION}/kics_${KICS_VERSION}_linux_x64.tar.gz" -O - | tar -xz --directory /usr/bin &>/dev/null
  script:
    - kics scan -q /usr/bin/assets/queries -p ${PWD} -o ${PWD}/kics-results.json
  artifacts:
    name: kics-results.json
    paths:
      - kics-results.json
```

- Runs only on merge request event.
- Download KICS binary to run the KICS SCAN.
- Store the output in a JSON format as artefacts. The output file name is `kics-results.json`.
---

#### KICS Scan Result

```YAML
kics-result:
  stage: kics-result
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
  before_script:
    - export TOTAL_SEVERITY_COUNTER=`grep '"total_counter"':' ' kics-results.json | awk {'print $2'}`
    - export SEVERITY_COUNTER_HIGH=`grep '"HIGH"':' ' kics-results.json | awk {'print $2'} | sed 's/.$//'`
    - export SEVERITY_COUNTER_MEDIUM=`grep '"INFO"':' ' kics-results.json | awk {'print $2'} | sed 's/.$//'`
    - export SEVERITY_COUNTER_LOW=`grep '"LOW"':' ' kics-results.json | awk {'print $2'} | sed 's/.$//'`
    - export SEVERITY_COUNTER_INFO=`grep '"MEDIUM"':' ' kics-results.json | awk {'print $2'} | sed 's/.$//'`
  script:
    - |
      echo "TOTAL SEVERITY COUNTER: $TOTAL_SEVERITY_COUNTER
      SEVERITY COUNTER HIGH: $SEVERITY_COUNTER_HIGH
      SEVERITY COUNTER MEDIUM: $SEVERITY_COUNTER_MEDIUM
      SEVERITY COUNTER LOW: $SEVERITY_COUNTER_LOW
      SEVERITY COUNTER INFO: $SEVERITY_COUNTER_INFO"
      if [ "$SEVERITY_COUNTER_HIGH" -ge "1" ];
        then echo "Please fix all $SEVERITY_COUNTER_HIGH HIGH SEVERITY ISSUES";
      fi
  dependencies:
    - "kics-scan"
```

- Runs only on merge request events and upon the successful completion of KICS SCAN.
- Parse the JSON file and retrieve the vulnerability results.
- Modify the stage exit condition as applicable. In this case, we use echo to show the number of high-severity counters. 

#### Terraform Validate

```YAML
validate:
  extends: .before_script_template
  stage: validate
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
      changes: 
        paths:
          - "terraform/**/*"
  script:
    - terraform validate
  dependencies:
    - "kics-scan"
```
- Runs on merge request events, and change is in any file underneath Terraform folder.

---
#### Terraform Plan

```YAML
plan:
  environment:
    name: $Environment
  rules:
    - if: $CI_COMMIT_BRANCH == "main" && $CI_PIPELINE_SOURCE == "push"
      changes: 
        paths:
          - "terraform/**/*"
  extends: .before_script_template
  stage: plan
  script:
    - |
      terraform plan \
      -var="resource_group_name=rgp-${Function_App_Name}-${Environment}" \
      -var="location=${location}" \
      -var="storage_account_name=${Function_App_Name}stg${Environment}" \
      -var="storage_account_tier=${storage_account_tier}" \
      -var="storage_account_replication_type=${storage_account_replication_type}" \
      -var="app_service_plan=${Function_App_Name}-${Environment}" \
      -var="app_service_plan_size=${app_service_plan_size}" \
      -var="app_service_plan_tier=${app_service_plan_tier}" \
      -var="function_app_name=${Function_App_Name}-${Environment}" \
      -out "plan.cache"
  artifacts:
    paths:
      - "terraform/plan.cache"
  dependencies:
    - validate
```

- Runs on merge request events, and change is in any file underneath Terraform folder.

---

#### Terraform Apply

```YAML
apply:
  environment:
    name: $Environment
  rules:
    - if: $CI_COMMIT_BRANCH == "main" && $CI_PIPELINE_SOURCE == "push"
      changes: 
        paths:
          - "terraform/**/*"
  extends: .before_script_template
  stage: apply
  artifacts:
    paths:
      - "terraform/plan.cache"
  script:
    - terraform apply -input=false plan.cache
  dependencies:
    - plan
```

- Apply to provision resources.
- Runs on merge request events, and change is in any file underneath Terraform folder.

--- 
#### Deploy Azure Functions

```YAML
deploy:
  environment:
    name: $Environment
  rules:
    - if: $CI_COMMIT_BRANCH == "main" && $CI_PIPELINE_SOURCE == "push"
      changes: 
        paths:
          - "src/**/*"
  stage: deploy
  image: python:3.9
  script:
    - pwd
    - apt-get update; apt-get install curl
    - curl -sL https://aka.ms/InstallAzureCLIDeb | bash
    - apt-get install curl && curl -sL https://deb.nodesource.com/setup_12.x | bash
    - apt-get install nodejs
    - npm install -g azure-functions-core-tools@4 --unsafe-perm true
    - az login --service-principal -u $ARM_CLIENT_ID -p $ARM_CLIENT_SECRET --tenant $ARM_TENANT_ID
    - func azure functionapp publish "${Function_App_Name}-${Environment}" --powershell --prefix src/
```

- Deploy Azure Functions when a change is in the SRC folder.
- Log in to Azure using AZ CLI.
- Publish Azure Functions using the Azure Functions core tools.


--- 

#### Az Functions Functional Test

```YAML
functionaltest:
  environment:
    name: $Environment
  rules:
    - if: $CI_COMMIT_BRANCH == "main" && $CI_PIPELINE_SOURCE == "push"
      changes: 
        paths:
          - "src/**/*"
  stage: functionaltest
  script:
    - apt-get update
    - apt-get install curl -y
    - |
      if [ "$(curl https://${Function_App_Name}-${Environment}.azurewebsites.net/api/iHome?name=Chen | grep Hello)" ]; then
        echo "success"
      else 
        exit 1
      fi
  dependencies:
    - "deploy"
```

- A simple CURL to check the accessibility of the API endpoint. 

---

#### Destroy

```YAML
destroy:
  when: manual
  environment:
    name: $Environment
  rules:
    - if: $CI_COMMIT_BRANCH == "main" && $CI_PIPELINE_SOURCE == "push"
  extends: .before_script_template
  stage: destroy
  script:
    - |
      terraform destroy \
      -var="resource_group_name=rgp-${Function_App_Name}-${Environment}" \
      -var="location=${location}" \
      -var="storage_account_name=${Function_App_Name}stg${Environment}" \
      -var="storage_account_tier=${storage_account_tier}" \
      -var="storage_account_replication_type=${storage_account_replication_type}" \
      -var="app_service_plan=${Function_App_Name}-${Environment}" \
      -var="app_service_plan_size=${app_service_plan_size}" \
      -var="app_service_plan_tier=${app_service_plan_tier}" \
      -var="function_app_name=${Function_App_Name}-${Environment}" \
      -auto-approve
```

- Destroy stage is to run on demand. 

### References

- [Azure Functions](https://azure.microsoft.com/en-gb/products/functions/#overview)
- [Azure Functions Core Tools](https://github.com/Azure/azure-functions-core-tools)
- [Terraform](https://www.terraform.io/)
- [GitLab Predefined Variables](https://docs.gitlab.com/ee/ci/variables/predefined_variables.html)
- [GitLab Terraform State](https://docs.gitlab.com/ee/user/infrastructure/iac/terraform_state.html)
### Summary

Congratulations, you have successfully deployed the serverless solution. Now, you know to create a CICD pipeline in GitLab to deploy and destroy Azure functions. In my upcoming blog post, I have plans to cover the below

:rocket: Implement a DevOps way to handle the enhancements.  
:rocket: Implement Azure Traffic Manager for high availability.  
:rocket: Blue Green Deployment.  


Your feedback is highly appreciated. Feel free to send your comments to my :envelope: [inbox](mailto:chendrayan.exchange@hotmail.com). 