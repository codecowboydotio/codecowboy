+++
title = "SQL Server on Kubernetes"
date = "2021-11-01"
aliases = ["sql_server"]
tags = ["kubernetes", "containers", "databases"]
categories = ["kubernetes", "software", "dev"]
[ author ]
  name = "codecowboy.io"
+++

# Intro
Recently, I've been working with a customer who wants to provide databases on their Kubernetes cluster. 
Ever since Microsoft's SQL Server was released on Linux some years ago, I've been fascinated with it.
I decided to give it a go recently on Kubernetes, and get it all working.


## Why databases?

## Why SQL server?

## Secret
In order to get the database up and running, you will need to have a secret. 
This is the initial SA password that is used for the database. 
The easiest way to do this is to create an opaque secret. 

The command below creates an opaque secret with a password that is complex enough to start the database.

```
kubectl create secret generic mssql --from-literal=SA_PASSWORD="MyC0m9l&xP@ssw0rd" --namespace=mssql
```

## Manifests
The manifests for deploying SQL server are relatively simple. 
The pages [here](https://docs.microsoft.com/en-us/sql/linux/sql-server-linux-overview?view=sql-server-ver15) give a really good overview of the general installation and command line options available for SQL Server on linux. These can be converted to manifest files.

### Namespace 
First we create a namespace. Technically, you don't need to do this and can run everything in the default namespace, but for neatness sake, I always think it's worth creating a separate namespace.

```
kind: Namespace
apiVersion: v1
metadata:
  name: mssql
  labels:
    name: mssql
```

### Pods
Create a deployment for SQL server. I am creating a deployment rather than a statefulset for demonstration purposes. 

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mssql-a
  namespace: mssql
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mssql-a
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: mssql-a
    spec:
      terminationGracePeriodSeconds: 10
      securityContext:
        fsGroup: 1000
      containers:
      - name: mssql
        image: mcr.microsoft.com/mssql/rhel/server:2019-latest
        ports:
        - containerPort: 1433
          name: mssql-port
          protocol: TCP
        env:
        - name: MSSQL_PID
          value: "Developer"
        - name: ACCEPT_EULA
          value: "Y"
        - name: MSSQL_SA_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mssql
              key: SA_PASSWORD
```

#### Environment variables 
The environment variables that can be used to configure MSSQL server are listed [here](https://docs.microsoft.com/en-us/sql/linux/sql-server-linux-configure-environment-variables?view=sql-server-ver15).

In the manifest above, I am using three variables.

- MSSQL_PID: The SQL Server edition or product key. In my case, "developer edition"
- ACCEPT_EULA: Accept the End User License Agreement
- MSSQL_SA_PASSWORD: The SA password for the database. In my case, this refers to the secret that I created earlier

### Service 
Create a service that can be used to expose the pods that we created above. The service is named **mssql-a** purely because I may have more than one database that i want to expose.

This service exposes the database pods on port 1433, the default SQL server port.

```
apiVersion: v1
kind: Service
metadata:
  name: mssql-a
  namespace: mssql
spec:
  selector:
    app: mssql-a
  ports:
    - protocol: TCP
      port: 1433
      targetPort: 1433
```

### Persistent Storage
I'll cover this piece in a second blog post, because it deserves its own topic entirely.

The database manifest works but will store data locally only. This means that it is only useful for development purposes. If the pod is restarted for any reason, data will be lost. 

## Client side tools
Install client side tools to connect to the database.

There is a really good document [here](https://docs.microsoft.com/en-us/sql/linux/quickstart-install-connect-red-hat?view=sql-server-ver15) that describes how to install the client side utilities in order to connect to your database.

I use fedora, so am using the instructions for RHEL8 (close enough)

Use curl to install the microsoft repository on your system.

```
sudo curl -o /etc/yum.repos.d/msprod.repo https://packages.microsoft.com/config/rhel/8/prod.repo
```

Install the client side tooling and the unix ODBC client
```
sudo yum install -y mssql-tools unixODBC-devel
```

Add the SQL tools to your default path and load the path into the current environment.
```
echo 'export PATH="$PATH:/opt/mssql-tools/bin"' >> ~/.bash_profile
echo 'export PATH="$PATH:/opt/mssql-tools/bin"' >> ~/.bashrc
source ~/.bashrc
```

Test your database.
```
sqlcmd -S localhost -U SA -P '<YourPassword>'
```
