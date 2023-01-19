---
author: "Chendrayan Venkatesan"
title: "Dapr state store demonstration using Minikube with a PowerShell app"
date: 2023-01-19
description: "this is one part of a microservice application I am building…"
tags: ["minikube", "powershell", "dapr"]
thumbnail: /2023/01/container-new.jpg
---

# Introduction

This blog post is a part of the feature I added to my web app built using PowerShell. I couldn’t hold the excitement. So, the share is not a minimum viable product but an idea to show a case running a Dapr as a sidecar in a pod. In my last blog post, I shared my understanding of Dockerizing full-stack application(s) built using PowerShell and ran using Docker-Compose and then over a Minikube. It works fine, and no complex steps are involved in it. Now, I need to introduce Dapr to simplify the state store management. How? We remove the SimplySQL module and enable Dapr as a Sidecar to run inside the pod. We use the Dapr state store component to insert the records into the Azure Table Storage.


### Disclaimer

- This blog post is the output of my learning on Docker, MySQL, docker-compose, and MiniKube. 
- I am an automation engineer with a decent experience in the cloud, not a software developer.
- We don’t talk about Pode, PSHTML & SimplySQL (PowerShell Modules).
- There is a big room for improvement. Consider this blog post as a first step to dockerize the PowerShell app to run with docker-compose and then on Minikube. 

### Prerequisites

- Dapr CLI
- Docker Desktop for Windows
- PowerShell
    - Pode
    - PSHTML
- MetroUI (Fontend toolkit)
- Minikube 
- Kubectl (optional) 

## Demo Use Case

- Let attendees fill in a form to register for an event. 
- Upon submitting the form
    - Store the information in the Azure Table.
    - Send a notification via Telegram. 
    - Web Response to convey thanks note for the registration.

## Solution

The abstract of our simple architecture is below 

![img](/2023/01/HLD-MS.png)

### Project Scaffolding

📦howdykloudy  
 ┣ 📂app  
 ┃ ┣ 📂views  
 ┃ ┃ ┣ 📜event-registration-response.ps1  
 ┃ ┃ ┣ 📜event-registration.ps1   
 ┃ ┃ ┗ 📜home.ps1  
 ┃ ┣ 📜app.ps1  
 ┃ ┗ 📜Dockerfile  
 ┣ 📂components  
 ┃ ┗ 📜statestore.yaml  
 ┣ 📂manifest  
 ┃ ┗ 📜deployment.yaml  
 ┗ 📜README.md  

- `app.ps1` is the entry point to start the web server
- `home.ps1` is the landing page. (home page)
- `event-registration.ps1` is the Get route to show the event registration form.
- `event-registration-response.ps1` is the post route for the state store (insert form information into the Azure Table Storage); send a telegram message and a web response (UI) to share a thank note for the registration.  

Please note that I don’t have a script/code to write the information to the state store. Instead, I have a state store component file defined in `statestore.yaml` file under the `components` directory. Below is the definition of the state store component. Yes, I have secrets in it. In my upcoming blogs, I plan to cover the secrets management in Dapr

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

replace the values for `{STORAGE-ACCOUNT-NAME}` , `{STORAGE-ACCOUNT-KEY}` and `{TABLE-NAME}`

### How does the state store component work? 

The state store component name is `eventregistrationresponse`. I have a route created with the same name (to avoid confusion), which invokes the REST API, as shown below

```PowerShell
Invoke-RestMethod -Uri "http://localhost:3500/v1.0/state/eventregistrationresponse" -Method Post -Body $($Body) -ContentType 'application/json' 
```

The [documentation](https://docs.dapr.io/developing-applications/building-blocks/state-management/state-management-overview/) gives more information on state management. 

### Telegram API for sending messages 

I need this feature, but I haven’t used the full feature part. Over here, I am showing a snippet to send a message when the registration form is submitted. 

```PowerShell
$Telegramtoken = "{API-TOKEN}"
$Telegramchatid = "{CHAT-ID}"
[Net.ServicePointManager]::SecurityProtocol = [Net.SecurityProtocolType]::Tls12
try {
    Invoke-RestMethod -Uri "https://api.telegram.org/bot$($Telegramtoken)/sendMessage?chat_id=$($Telegramchatid)&text=$($Body)"
    Write-PodeHost -Object "sending telegram notification..."
}
catch {
    Write-PodeHost -Object $($_.Exception.Response)
} 
```

> Follow the [link](https://polarclouds.co.uk/send-telegram-from-powershell/) to get the API token and chat ID. 

### Get on Action 

First, let me show you the steps to run the solution locally. 

```PowerShell
PS D:\projects\howdykloudy> dapr uninstall
PS D:\projects\howdykloudy> dapr init --runtime-version 1.0.0-rc.3
PS D:\projects\howdykloudy> dapr run --log-level debug --app-id 'eventregistration' --app-port 3000 --dapr-http-port 3500 --dapr-grpc-port 60002 --components-path .\components -- pwsh .\app\app.ps1
```

- `dapr uninstall` - Remove any previous configuration to be on the safer side. 
- `dapr init --runtime-version 1.0.0-rc.3` - Initialize the Dapr with the runtime version. I couldn’t make it work without it.  
- `dapr run --log-level debug --app-id 'eventregistration' --app-port 3000 --dapr-http-port 3500 --dapr-grpc-port 60002 --components-path .\components -- pwsh .\app\app.ps1` - Run the application in debug mode to see the message on the host; the app id is your choice, the app port is the same as the port defined in your application, the dapr HTTP port we use for the state management API and also the gRPC port. 


Without a doubt, I see the result!  

![img](/2023/01/State-Store.png)

Now let us take a few more steps to run the app on Minikube. For this, I have taken a few baby steps 

#### Create a local registry 

```PowerShell
PS D:\projects\howdykloudy> docker run -d -p 5000:5000 --restart=always --name registry registry:2
```

#### Build & Tag a Image

```PowerShell
PS D:\projects\howdykloudy\app> docker build -t localhost:5000/podeux:v1 .
```

#### Push the image to the local registry

```PowerShell
PS D:\projects\howdykloudy\app> docker push localhost:5000/podeux:v1
```

#### Magic

Gear up to run the Minikube, and the below snippet does the magic.  

```PowerShell
minikube start
dapr init --runtime-version 1.0.0-rc.3 --kubernetes 
minikube image load localhost:5000/podeux:v1
```

- `minikube start` - is to start the Minikube cluster
- `dapr init --runtime-version 1.0.0-rc.3 --kubernetes` - initialize the dapr to run on Minikube cluster
- `minikube image load localhost:5000/podeux:v1` - load the image to Minikube (The image we pushed to the local registry - podeux:v1)

```PowerShell
kubectl apply -f .\components\statestore.yaml
kubectl apply -f .\manifest\deployment.yaml 
kubectl rollout status deploy/podeux
kubectl port-forward service/podeux 3000:3000
```

- `kubectl apply -f .\components\statestore.yaml` - Creates the state store
- `kubectl apply -f .\manifest\deployment.yaml ` - Creates deployment and service 
- `kubectl rollout status deploy/podeux` - Kubernetes deployments are asynchronous, meaning you'll need to wait for the deployment to complete before moving on to the next steps. You can do so with the following command:
- `kubectl port-forward service/podeux 3000:3000` - There are several different ways to access a Kubernetes service, depending on which platform you are using. Port forwarding is one consistent way to access a service, whether the workload is hosted locally or on a cloud Kubernetes provider like AKS.

![TM](/2023/01/TM.png)

## Summary

Thanks, it took 2 hours for me to connect the dots and make it happen. I hope this may help someone there to amalgamate the PowerShell with Dapr and run a microservice application on a Minikube cluster. Yes, we are there! Soon we will spin up our app on the cloud. Stay tuned. 