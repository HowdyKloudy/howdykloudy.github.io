---
author: "Chendrayan Venkatesan"
title: "Dockerize the Full-Stack application built using PowerShell (Pode & PSHTML)"
date: 2023-01-12
description: ""
tags: ["minikube", "powershell", "docker-compose", ""]
thumbnail: /2023/01/container.jpg
---

## Introduction

I have recorded the demonstration of spinning up the full-stack application built using PowerShell (Pode & PSHTML). Feel free to leave your comments 

{{< youtube DZvLmIUGNno>}}

### Disclaimer

- This blog post is the output of my learning on Docker, MySQL, docker-compose, and MiniKube. So, please skip asking ask why not native or alternatives. I am sharing it as a blog because of the ask that I have seen it in the [REDDIT](https://www.reddit.com/r/PowerShell/comments/gq0bl0/discussion_can_powershell_be_used_for_fullstack/).
- I am an automation engineer with a decent experience in the cloud, not a software developer. 
- We don't talk about Pode, PSHTML & SimplySQL (PowerShell Modules) 
- There is a big room for improvement. Consider this blog post as a first step to dockerize the PowerShell app to run with docker-compose and then on Minikube. 

### Prerequisites

- Docker Desktop for Windows
- PowerShell
    - Pode
    - PSHTML
    - SimplySQL
- MetroUI (Fontend toolkit)
- Minikube 
- Kubectl (optional) 

## Run a Full-Stack Application Locally (Windows Box)

We aren't building an application with real-time data that visually pleases the end users. Instead, we intend to collect and store the employee information in the MySQL table. So, let us build a simple page showing useless data and a form to get employee information and insert it in the employee table as a row with an auto incremental employee id. 

### High-Level Diagram

![Ful-Stack](/2023/01/FS.png)

## Solution

First, decide the data you need to collect and store in the MySQL table. In my case, I need to get the user's first name, last name, city, country and mobile number. 

### Project Structure

📦howdykloudy  
 ┣ 📂app  
 ┃ ┣ 📂views  
 ┃ ┃ ┗ 📜index.ps1  
 ┃ ┣ 📜app.ps1  
 ┃ ┗ 📜Dockerfile  
 ┣ 📂dump  
 ┃ ┗ 📜dump.sql  
 ┣ 📜docker-compose.yml  
 ┣ 📜minikube-deployment.yml  
 ┗ 📜README.md  

- `howdykloudy` is the project root folder.
- `app` is the `folder` to store the user interface code.
- `app\views` to hold the `index.ps1` file.
- `app\app.ps1` is the code to start the webserver.
- `Dockerfile` – is to dockerize the user interface. 
- `dump/dump.sql` – Is the file to create a database table
- `docker-compose.yml` - Is the file is to bring up two interconnected containers. 

### Script to Create a MYSQL Table

```sql
create table employee (
    id int not null AUTO_INCREMENT primary key,
    first_name varchar(255),
    last_name varchar(255),
    city varchar(255),
    country varchar(255),
    mobile varchar(255)
)
```

`id int not null AUTO_INCREMENT primary key` - For each record insertion, we see the ID value incremented by 1. For example, the first record in the database looks like below 

| ID   | FirstName | LastName | City      | Country | Mobile         |
| ---- | --------- | -------- | --------- | ------- | -------------- |
| 1    | Chen      | V        | Bengaluru | India   | +91 0123456789 |

and the subsequent calls continue to store records as illustrated below 

| ID   | FirstName | LastName | City      | Country | Mobile         |
| ---- | --------- | -------- | --------- | ------- | -------------- |
| 1    | Chen      | V        | Bengaluru | India   | +91 0123456789 |
| 2    | Chen      | V        | Bengaluru | India   | +91 0123456789 |

### User Interface (Web Front-End)

```PowerShell
#region - Form
form -action "/formdata" -method "post" -enctype 'multipart/form-data' -content {
    Div -Class 'form-group' -Content {
        h5 -Content 'Employee Form' -Style 'text-align:center'
        input -type text -name 'first_name' -Attributes @{'data-role' = "materialinput"; 'data-icon' = "<span class=mif-first>"; 'data-cls-line' = 'bg-darkCyan'; 'data-cls-label' = 'fg-darkCyan'; 'data-cls-informer' = 'fg-darkCyan'; 'data-cls-icon' = 'fg-darkCyan' ; 'placeholder' = 'First Name' } -required
        input -type text -name 'last_name' -Attributes @{'data-role' = "materialinput"; 'data-icon' = "<span class=mif-last>"; 'data-cls-line' = 'bg-darkCyan'; 'data-cls-label' = 'fg-darkCyan'; 'data-cls-informer' = 'fg-darkCyan'; 'data-cls-icon' = 'fg-darkCyan' ; 'placeholder' = 'Last Name' } -required
        input -type text -name 'city' -Attributes @{'data-role' = "materialinput"; 'data-icon' = "<span class=mif-location-city>"; 'data-cls-line' = 'bg-darkCyan'; 'data-cls-label' = 'fg-darkCyan'; 'data-cls-informer' = 'fg-darkCyan'; 'data-cls-icon' = 'fg-darkCyan' ; 'placeholder' = 'City' } -required
        input -type text -name 'country' -Attributes @{'data-role' = "materialinput"; 'data-icon' = "<span class=mif-flag>"; 'data-cls-line' = 'bg-darkCyan'; 'data-cls-label' = 'fg-darkCyan'; 'data-cls-informer' = 'fg-darkCyan'; 'data-cls-icon' = 'fg-darkCyan' ; 'placeholder' = 'Country' } -required
        input -type tel -name 'mobile' -Attributes @{'data-role' = "materialinput"; 'data-icon' = "<span class=mif-mobile>"; 'data-cls-line' = 'bg-darkCyan'; 'data-cls-label' = 'fg-darkCyan'; 'data-cls-informer' = 'fg-darkCyan'; 'data-cls-icon' = 'fg-darkCyan' ; 'placeholder' = 'Mobile' } -required
    }
    div -class 'form-group' -content {
        button -class 'button bg-darkCyan outline rounded fg-white' -content 'Submit'
    }
}
#endregion
```

{IMAGE}

> On Submit (Insert Records into MySQL)

## Docker Compose

Docker Compose is a tool to run multi-container Docker applications. Below is the YAML definition 

```YAML
version: '3.8'
networks:
  howdy_kloudy:
    name: howdy_kloudy
services:
  mysql:
    image: mysql
    restart: always
    container_name: db-mysql
    networks:
      - howdy_kloudy
    ports:
      - 3306:3306
    environment:
      - MYSQL_DATABASE=local_db
      - MYSQL_ALLOW_EMPTY_PASSWORD=yes
    volumes:
      - ./dump:/docker-entrypoint-initdb.d

  podeui:
    build: ./app
    image: podeui
    networks:
      - howdy_kloudy
    ports:
      - "3000:3000"

```

- `mysql` - use the latest `mysql` image. 
- `podeui` - Build path ./app and image name is `podeui`
- `networks` - Instead of just using the default app network, you can specify your own networks with the top-level networks key (howdy_kloudy). This lets you create more complex topologies and specify custom network drivers and options. You can also use it to connect services to externally-created networks which aren’t managed by Compose. (I need this for my future use case)

```PowerShell
PS C:\Projects\howdykloudy> docker-compose up --build --detach
```

![Images In Use](/2023/01/OP1.png)

![Images In Use](/2023/01/OP2.png)

Now access the application from the local host. 

> UI

![UI](/2023/01/OP3.png)

> Response 

![Response](/2023/01/OP4.png)

Refer the records in the MySQL table 

![First Record](/2023/01/OP5.png)

Second record

![Second Record](/2023/01/OP6.png)

Okay, it's working as expected! We see records are available in the table. 

## Run on Minikube

Nothing interesting, right? Let us add spice by running our application on Minikube. I spent a few hours establishing the connection between containers in a POD. Yeah, I know it's simple and easy, but for me, it's not. I failed to establish the connection between my UI and DB. 


```YAML
apiVersion: apps/v1
kind: Deployment
metadata:
  name: podeui
  labels:
    app: multi-contianer
spec:
  replicas: 1
  selector:
    matchLabels:
      app: multi-contianer
  template:
    metadata:
      labels:
        app: multi-contianer
    spec:
      containers:
        - name: podeui
          image: chenv/podeui:v3
          ports:
            - containerPort: 3000
          resources: {}
        - name: db-mysql
          image: mysql
          env:
            - name: MYSQL_DATABASE
              value: "local_db"
            - name: MYSQL_ALLOW_EMPTY_PASSWORD
              value: "yes"
          resources: {}
          ports:
            - containerPort: 3306
          volumeMounts:
            - name: mysql-initdb
              mountPath: /docker-entrypoint-initdb.d
            - mountPath: "/var/lib/mysql"
              subPath: "mysql"
              name: mysql-data
      volumes:
        - name: mysql-initdb
          configMap:
            name: mysql-initdb-config
        - name: mysql-data
          persistentVolumeClaim:
            claimName: mysql-data-disk
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: mysql-initdb-config
data:
  init.sql: |
    CREATE DATABASE IF NOT EXISTS local_db;
    USE local_db;
    create table employee (id int not null AUTO_INCREMENT primary key, first_name varchar(255), last_name varchar(255), city varchar(255),country varchar(255), mobile varchar(255));
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-data-disk
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

Yes, the YAML above is self-explanatory. Just a few highlights

- `ConfigMap` : A ConfigMap is not to hold large chunks of data. The data stored in a ConfigMap cannot exceed 1 MiB. If you need to store settings larger than this limit, you may want to consider mounting a volume or using a separate database or file service.
- `PersistentVolumeClaim` : A PersistentVolumeClaim (PVC) is a request for storage by a user. It is similar to a Pod. Pods consume node resources, and PVCs consume PV resources. Pods can request specific levels of resources (CPU and Memory). Claims can request size and access modes (e.g., they can be mounted ReadWriteOnce, ReadOnlyMany or ReadWriteMany, see AccessModes). While PersistentVolumeClaims allow a user to consume abstract storage resources, it is expected that users need persistent volumes with varying properties, such as performance, for different problems. 

## Summary

I plan to extend this application and share my Kubernetes learnings. So, stay tuned to get to know more on the below 

- Nuances of Minikube. 
- Tour of Kubectl
- and more…..
