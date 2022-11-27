---
title: "GITLAB CICD TO DEPLOY AZURE FUNCTIONS WITH GITLAB"
date: 2022-11-27T20:32:21Z

draft: false
tags: ["GitLab", "KICS", "Azure" , "Azure Functions"]
type: post
showTableOfContents: true
---

![img](/images/avatar.png)

## Introduction

Hello, DevOps Focals! How have you been? It’s my pleasure to share the learnings in GitLab I have gained in the recent past. A month ago, I demonstrated deploying Azure Functions to a project team, and this blog post is to walk through the steps

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

 ```Terraform 
terraform {
  backend "http" {
    address  = "${TF_ADDRESS}"
    username = "${TF_USERNAME}"
    password = "${TF_PASSWORD}"
  }
}
 ```