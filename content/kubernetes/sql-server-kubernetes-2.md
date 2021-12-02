+++
title = "SQL Server on Kubernetes - Part 2"
date = "2021-12-01"
aliases = ["sql_server"]
tags = ["kubernetes", "containers", "databases"]
categories = ["kubernetes", "software", "dev"]
[ author ]
  name = "codecowboy.io"
+++

# Intro

So in my last [post](https://codecowboy.io/kubernetes/sql-server-kubernetes/) I showed how you could create databases on Kubernetes. There are many reasons to do this. Equally, there are reasons not to do this, but for highly distributed deployments it does make sense.

This post is going to focus on the storage components of running a database on Kubernetes.

## Why do I need persistent storage

Persistent storage as the name implies allows you to store your data between container restarts. This is important where data is stored in a database. You actually want your data to be persisted. 

## How do I do this in kubernetes?

Kubernetes has two concepts that allow you to persist data that we are going to use. The first is a persistent volume claim, the second is a persistent volume mount

## Persistent Volume Claim

In Kubernetes a persistent volume claim is a way of "grabbing some storage" if you're a user. The Kubernetes documentation describes this much more eloquently as "a request for storage from a user" and "the persistent volume API abstracts the details of storage from the user".

This is documented [here](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)

In order to create a persistent volume claim, you can use a manifest like the one below.

 
```Yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mssql-data
  namespace: mssql
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 100Mi
  volumeMode: Filesystem
```

This will request 100M as a filesystem mount with a Read and Write access mode, within the namespace mssql.

Once I have a claim, I can create a pod that utilises the claim, and mounts a filesystem.


## Persistent Storage Volumes

Pods can use a claim as a volume. In order to do this, the following is added to the Deployment definition.

```Yaml
        volumeMounts:
        - name: mssqldb
          mountPath: /var/opt/mssql
      volumes:
      - name: mssqldb
        persistentVolumeClaim:
          claimName: mssql-data
```

This mounts the volume by mapping to a claim name.

Note that the claim name is the same as our claim name in our PersistentVolumeClaim definition above. This essentially makes the "raw disk" available to the pod.

The volume mount, mounts the volume as a filesystem within the container or pod. The reason that I can do this is that the volume mode in my PersistentVolumeClaim is set to Filesystem. This allows the volume to be mounted as a filesystem within the container / pod.

The full definition is below.

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
        volumeMounts:
        - name: mssqldb
          mountPath: /var/opt/mssql
      volumes:
      - name: mssqldb
        persistentVolumeClaim:
          claimName: mssql-data
```


## Inside the container

Inside the container I see the following

```Bash
[root@fedora mssql]# kubectl exec -it mssql-a-59b4fbc56d-68rzj -n mssql /bin/bash

bash-4.4$
bash-4.4$ df -h
Filesystem      Size  Used Avail Use% Mounted on
overlay          80G  4.3G   76G   6% /
tmpfs            64M     0   64M   0% /dev
tmpfs           7.9G     0  7.9G   0% /sys/fs/cgroup
/dev/xvda1       80G  4.3G   76G   6% /etc/hosts
shm              64M     0   64M   0% /dev/shm

/dev/xvdca      976M  112M  849M  12% /var/opt/mssql

tmpfs           7.9G   12K  7.9G   1% /run/secrets/kubernetes.io/serviceaccount
tmpfs           7.9G     0  7.9G   0% /proc/acpi
tmpfs           7.9G     0  7.9G   0% /proc/scsi
tmpfs           7.9G     0  7.9G   0% /sys/firmware

```

You can see that the /var/opt/mssql filesystem is a filesystem that I can put data into.
In my case, this is the default data directory for my SQL Server database.



## What if the pod restarts?

## Conclustion

ZZ
