+++
title = "SQL Server on Kubernetes - Part 1"
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

This is part one, where I deploy SQL server without persistent storage. 
In part two, I will discuss using persistent storage.

## Why databases?
There is a lot of debate about whether or not you *should* run databases on kubernetes or not. If you're operating in a public cloud environment, this is much more clear cut to my mind than if you're not. If you are, then it may be better to use a service from a cloud provider where infrastructure is taken care of for you. It's just easier.

If you are not operating in a public cloud environment, then running on kubernetes gives you the resilience and abstraction from infrastructure that is as close as you can get to running in a public cloud. This is very useful in disconnected environments and environments where you cannot access public cloud (yes they do exist).

Suffice to say, there are reasons that you may want to do this.

## Why SQL server?

SQL server is ubiquitous. It is the database that a lot of applications use. As applications get either refactored or shifted to kubernetes, it is reasonable to assume that there will be instances where running a SQL server database on kubernetes is needed.

## Secret
In order to get the database up and running, you will need to have a secret. 
This is the initial SA password that is used for the database. 
The easiest way to do this is to create an opaque secret. 

The command below creates an opaque secret with a password that is complex enough to start the database.

```Bash
kubectl create secret generic mssql --from-literal=SA_PASSWORD="MyC0m9l&xP@ssw0rd" --namespace=mssql
```

## Manifests
The manifests for deploying SQL server are relatively simple. 
The pages [here](https://docs.microsoft.com/en-us/sql/linux/sql-server-linux-overview?view=sql-server-ver15) give a really good overview of the general installation and command line options available for SQL Server on linux. These can be converted to manifest files.

### Namespace 
First we create a namespace. Technically, you don't need to do this and can run everything in the default namespace, but for neatness sake, I always think it's worth creating a separate namespace.

```Yaml
kind: Namespace
apiVersion: v1
metadata:
  name: mssql
  labels:
    name: mssql
```

### Pods
Create a deployment for SQL server. I am creating a deployment rather than a statefulset for demonstration purposes. 

```Yaml
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

```Yaml
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

```Bash
sudo curl -o /etc/yum.repos.d/msprod.repo https://packages.microsoft.com/config/rhel/8/prod.repo
```

Install the client side tooling and the unix ODBC client
```Bash
sudo yum install -y mssql-tools unixODBC-devel
```

Add the SQL tools to your default path and load the path into the current environment.
```Bash
echo 'export PATH="$PATH:/opt/mssql-tools/bin"' >> ~/.bash_profile
echo 'export PATH="$PATH:/opt/mssql-tools/bin"' >> ~/.bashrc
source ~/.bashrc
```

Test your database.
```Bash
sqlcmd -S localhost -U SA -P '<YourPassword>'
```

## Port forward from local machine to database
As I have not created any ingress for my database, the easiest way for me to get connectivity is to port forward directly to it.
I can use the command below to port forward from my local workstation to my database.

First I need to get the pod name of my database in order to port forward to it.
```Bash
kubectl get pods -n mssql

NAME                       READY   STATUS    RESTARTS   AGE
mssql-a-8469f884f7-rrbx9   1/1     Running   0          18m
```

I can then use the port-forward command to forward a local port to the pod port so that I can perform some testing and check that my database actually works.

```Bash
kubectl port-forward mssql-a-<pod> 1433:1433 -n mssql --address 0.0.0.0
```

## Database connect and test
Once everything has been created on the kubernetes side of the house, we can connect to the database and see that it is available.

I can connect to my database using the password I set originally. As I have port forwarded to my cluster, no ingress is needed. This is useful for testing.

I create a database named **foo**

```Bash
[root@fedora]# sqlcmd -S localhost -U SA -P 'MyC0m9l&xP@ssw0rd'
1> create database foo
2> go
```

If I select the names of all databases from the sys.Database table, I can see that the last entry is my database **foo**.
```sql
1> select name from sys.Databases
2> go
name
--------------------------------------------------------------------------------------------------------------------------------
master
tempdb
model
msdb
foo

(5 rows affected)
```

I can switch to the **foo** database and being to use it.
I create a table and insert a single line of data into my newly created database.

```sql
1> use foo
2> go
Changed database context to 'foo'.

1> create table bar (id INT, name VARCHAR(50))
2> go

1> insert into bar values (1, 'test')
2> go

(1 rows affected)
```

If I select all of the data from my table **bar** I can see the single line of data that I inserted above. 
```sql
1> select * from bar
2> go
id          name
----------- --------------------------------------------------
          1 test
```

I have a functional database that is running on kubernetes!


## Conclusion
Running databases on kubernetes isn't that difficult. There are reasons that you want to do this.
The difficult part about this is the ephemeral nature of pods on kubernetes and how to handle persistent storage with databases.
This is the topic of my next post, where I will show how to use persistent storage to make your databases on kubernetes more robust.

