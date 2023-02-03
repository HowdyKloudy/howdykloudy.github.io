---
author: "Chendrayan Venkatesan"
title: "Dapr Secret Store Component Demonstration Using Minikube With a PowerShell App"
date: 2023-01-29
description: "security is everybody's business..."
tags: ["dapr", "secret-store","minikube", "Azure"]
thumbnail: /2023/01/Secure-Key.jpg
---

# Introduction

The content curation process for this blog post exhausted me, and the reason is a distraction. Read and enjoy the funny story! I made a lot of trial and error before sharing it with you all. My previous blog post discussed using Dapr on Minikube with a PowerShell app. You can refer to the link below to learn about the use case of state store component

- [Dockerize the Full-Stack application built using PowerShell (Pode & PSHTML)](https://howdykloudy.github.io/blog/dockerize-the-full-stack-application-built-using-powershellpode-pshml/)
- [Dapr state store demonstration using Minikube with a PowerShell app](https://howdykloudy.github.io/blog/dapr-state-store-demonstration-using-minikube-with-a-powershell-app/)

## Fix the gap in handling secrets

I hope you read my previous blogs. If not, it's okay! Here is the problem for which we get a fix.

> Secrets in the YAML are bad - Key Vault

```YAML
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: azurekeyvault
spec:
  type: secretstores.azure.keyvault
  version: v1
  metadata:
  - name: vaultName
    value: "kv-howdykloudy-dev"
  - name: azureTenantId
    value: "d15f83d0-ed59-4e08-925a-e7445f64efe8"
  - name: azureClientId
    value: "9d5b1b0c-4a87-45b1-a012-de2d83e81ce8"
  - name: azureClientSecret
    value: "{YOUR-SUPER-SECRET}"
```

> Secret Store

```YAML
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: eventregistrationresponse
spec:
  type: state.azure.tablestorage
  version: v1
  metadata:
    - name: accountName
      value: {STORAGE-ACCOUNT-NAME}
    - name: accountKey
      value: {STORAGE-ACCOUNT-KEY}
    - name: tableName
      value: {TABLE-NAME}
    - name: cosmosDbMode
      value: true    
```

I have secrets in YAML and also in the app. How to handle it securely? 

![App Secrets](/static/2023/01/L3.png)

### What is covered in this blog post? 

- Remove the clear text credentials/secrets from the YAML file. Pass the parameters to authenticate with Azure table storage through the **kubectl command line.** 
- Securely access the application
- Store the app secrets in the **Azure key vault**. I have secrets that need for the application
    - For example, the telegram API key and chat ID to post the notification

I value your time and recorded the Dapr secret store component demo using Minikube with a PowerShell app. Feel free to share your comments and feedback to improvise the content are most welcome 

{{<youtube zTp6Aj-2LJI>}}

# About the Secret Store Component

The Dapr Secret Store component is a feature in Dapr that enables you to store, manage, and retrieve secrets, such as passwords, keys, and certificates, in a secure and scalable manner. The Dapr Secret Store component is designed to be used in a microservices architecture, allowing you to independently store and manage secrets for each microservice.

**Here are some key features of the Dapr Secret Store component:**

- Decentralized: The Dapr Secret Store component is decentralized, meaning that secrets are stored and managed locally on each node in a microservices architecture which reduces the risk of data breaches and makes it easier to manage secrets in large and complex microservices environments.
- **Secure:** The Dapr Secret Store component uses encryption and other security measures to protect secrets, ensuring that unauthorized parties cannot access them.
- **Scalable:** The Dapr Secret Store component is designed to scale horizontally, meaning it can easily handle multiple secrets and quickly deploy them in large-scale microservices environments.
- **API-based:** The Dapr Secret Store component provides a simple API that enables you to store, retrieve, and manage secrets in your microservices application. Your microservices can use this API to access secrets as needed.
- **Integration with other Dapr components:** The Dapr Secret Store component integrates with other Dapr components, such as the Dapr Configuration component, allowing you to manage secrets and configuration settings in a centralized and consistent manner.

# Solution

This time, we only focus on running the PowerShell web application on the Minikube cluster and not locally! The image below depicts the high-level logical flow 

![high-level](/2023/01/HLD-MS.png)

Deploy a pod with a web application and inject a Dapr sidecar. The Dapr sidecar's responsibility is to securely retrieve secrets from the Azure Key vault and communicate with Azure table storage. I know it's much theory. Let me walk through the steps 

## Azure App Registration With Certificate

You can use the existing service principal with a certificate or secret key. Follow the steps if you need a new SPN with a certificate. 

> You can use Az CLI, PowerShell or REST API. 

```PowerShell
# Create a resource group
az group create -n "{RESOURCE-GROUP}" --location "{LOCATION}"

# Create a key vault
az keyvault create --location "{LOCATION}" --name "{KEY-VAULT-NAME}" --resource-group "{RESOURCE-GROUP}"

# Create a service principal and configure its access to Azure resources.
az ad sp create-for-rbac --name "{SPN-NAME}" --create-cert --cert "{CERT-NAME}" --keyvault "{KEY-VAULT-NAME}" --skip-assignment --years 1

# Set Key Vault Policy
az keyvault set-policy --name "{KEY-VAULT-NAME}" --object-id "{SPN-OBJECT-ID}" --secret-permissions get

# Download and store the PFX in a secure location
az keyvault secret download --vault-name "{KEY-VAULT-NAME}" --name "{CERT-NAME}" --encoding base64 --file "{CERT-NAME}.pfx"

```

## Set up a local Docker registry

```PowerShell
docker run -d -p 5000:5000 --restart=always --name registry registry:2
```

## Docker Build & Tag

```PowerShell
docker build -t localhost:5000/podeux:v1 . --no-cache
```

## Push the image to the local registry

```PowerShell
docker push localhost:5000/podeux:v1
```

## Work with Minikube

```PowerShell
# Start the Minikube
minikube start --vm-driver=hyperv

# Create a secret using kubectl - This is used in kubernetes
kubectl create secret generic dapr-cert --from-file=daprpcert.pfx

# Initiate the Dapr inside the Minikube cluster (kubernetes)
dapr init --runtime-version 1.0.0-rc.3 --kubernetes 

# Load the image (local registry)
minikube image load localhost:5000/podeux:v1 --overwrite=true
```

## Explore the infrastructure

### List the pods

```PowerShell
kubectl get pods -A
```

### Access the dashboard

```PowerShell
# access the minikube dashboard
minikube dashboard

# access the dapr dashboard
dapr dashboard -k
```

## Deploy the state store component

```YAML
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: azurekeyvault
  namespace: default
spec:
  type: secretstores.azure.keyvault
  version: v1
  metadata:
  - name: vaultName
    value: kv-dapr-dev
  - name: spnTenantId
    value: "d15f83d0-ed59-4e08-925a-e7445f64efe8"
  - name: spnClientId
    value: "a4196419-e629-4d3b-95a0-39698d9750db"
  - name: spnCertificate
    secretKeyRef:
      name: dapr-cert
      key:  daprpcert.pfx
auth:
    secretStore: kubernetes  
```

```PowerShell
kubectl apply -f .\components\azurekeyvault.yaml
```

## Deploy the App & Service

```YAML
kind: Service
apiVersion: v1
metadata:
  name: podeux
  labels:
    app: podeux
spec:
  selector:
    app: podeux
  ports:
    - port: 3000
---
kind: Deployment
apiVersion: apps/v1
metadata:
  name: podeux
  labels:
    app: podeux
spec:
  replicas: 1
  selector:
    matchLabels:
      app: podeux
  template:
    metadata:
      labels:
        app: podeux
      annotations:
        dapr.io/enabled: "true"
        dapr.io/app-id: "podeux"
        dapr.io/app-port: "3000"
        dapr.io/log-as-json: "true"
        dapr.io/enable-debug: "true"
    spec:
      containers:
        - name: podeux
          image: localhost:5000/podeux:v1
          imagePullPolicy: Never
          ports:
            - containerPort: 3000
          resources: {}
```

```PowerShell
kubectl apply -f .\manifest\deployment.yaml
```

## Rollout 

```PowerShell
# To see the Deployment rollout status, run kubectl rollout status
kubectl rollout status deploy/podeux
```

## Expose the Service

```PowerShell
# Access the app through port 3000 (both app & target port in 3000)
kubectl port-forward service/podeux 3000:3000

# Alternative - Access the app through port 3500 (localhost)
kubectl port-forward service/podeux 3500:3000
```

Now, access the application from your local host - http://localhost:3000 

Click on telegram button, the below code does the magic (access the secrets from KV through the Dapr API)

```PowerShell
#region - Telegram
Add-PodeRoute -Method Get -Path '/telegram' -ScriptBlock {
    $Telegramtoken = (Invoke-RestMethod -Uri 'http://localhost:3500/v1.0/secrets/azurekeyvault/Telegramtoken' -ContentType 'application/json').Telegramtoken
    $Telegramchatid = (Invoke-RestMethod -Uri 'http://localhost:3500/v1.0/secrets/azurekeyvault/Telegramchatid' -ContentType 'application/json').Telegramchatid
    [Net.ServicePointManager]::SecurityProtocol = [Net.SecurityProtocolType]::Tls12
    try {
        $Body = [PSCustomObject]@{
            Message = 'Secure Message Test'
        } | ConvertTo-Json -Compress
        Invoke-RestMethod -Uri "https://api.telegram.org/bot$($Telegramtoken)/sendMessage?chat_id=$($Telegramchatid)&text=$($Body)"
    }
    catch {
        Write-PodeHost -Object $($_.Exception.Response)
    } 
}
#endregion
```

To insert records, the below code calls Dapr API to communicate with Table Storage

```PowerShell
$employeeId = $([string]::Concat($(Get-Random -Minimum 50), '-', 'empid'))
$key = $($employeeId)
$data = [PSCustomObject]@{
    first_name = $($WebEvent.Data.first_name)
    last_name  = $($WebEvent.Data.last_name)
}
$State = @(
    [ordered]@{
        key   = $key
        value = $($data)
    }
)
$Body = ConvertTo-Json -InputObject $State -Compress
Write-PodeHost -Object $($Body)
Invoke-RestMethod -Uri "http://localhost:3500/v1.0/state/eventregistrationresponse" -Method Post -Body $($Body) -ContentType 'application/json' -Verbose -ErrorAction Stop
```

# References

- [Azure Key Vault secret store](https://docs.dapr.io/reference/components-reference/supported-secret-stores/azure-keyvault/)
- [Secrets management overview](https://docs.dapr.io/developing-applications/building-blocks/secrets/secrets-overview/ )

# Summary

The Dapr Secret Store component is a powerful tool for managing secrets in microservices applications, providing security, scalability, and ease of use. An exciting and essential feature, right? Go for it! Invest your time in building Darized / Dockerized apps and run them on Kubernetes.  