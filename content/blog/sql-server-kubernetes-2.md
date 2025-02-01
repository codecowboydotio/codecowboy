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

## How do I check this?

If you want to check a persistent volume claim use the following commands.
*kubectl get pvc* will show the current persistent volume claims and their status.

For each persistent volume, you should be able to see the mode, capacity, storageclass and so on.

```Bash
[root@fedora mssql]# kubectl get pvc -n mssql
NAME         STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
mssql-data   Bound    pvc-5cd23b78-9feb-4db0-b2b0-dca7f6b56371   1Gi        RWO            gp2            33h
```

Further information can be gained by using the describe key word.


```Bash
[root@fedora mssql]# kubectl describe  pvc -n mssql
Name:          mssql-data
Namespace:     mssql
StorageClass:  gp2
Status:        Bound
Volume:        pvc-5cd23b78-9feb-4db0-b2b0-dca7f6b56371
Labels:        <none>
Annotations:   pv.kubernetes.io/bind-completed: yes
               pv.kubernetes.io/bound-by-controller: yes
               volume.beta.kubernetes.io/storage-provisioner: kubernetes.io/aws-ebs
               volume.kubernetes.io/selected-node: ip-192-168-44-199.ap-southeast-2.compute.internal
Finalizers:    [kubernetes.io/pvc-protection]
Capacity:      1Gi
Access Modes:  RWO
VolumeMode:    Filesystem
Used By:       mssql-a-59b4fbc56d-68rzj
Events:        <none>
```

In order to see the actual volumes you can use the following commands.

This shows the volume, it's status and storage class, as well as the associated volume claim, including the namespace that the volume claim lives in.

```Bash
[root@fedora mssql]# kubectl get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM              STORAGECLASS   REASON   AGE
pvc-5cd23b78-9feb-4db0-b2b0-dca7f6b56371   1Gi        RWO            Delete           Bound    mssql/mssql-data   gp2                     33h
```

Similarly, when I perform a describe against the volume, we can see that this is running in AWS, we can see the type of storage and so on.


```Bash
[root@fedora mssql]# kubectl describe pv
Name:              pvc-5cd23b78-9feb-4db0-b2b0-dca7f6b56371
Labels:            failure-domain.beta.kubernetes.io/region=ap-southeast-2
                   failure-domain.beta.kubernetes.io/zone=ap-southeast-2c
Annotations:       kubernetes.io/createdby: aws-ebs-dynamic-provisioner
                   pv.kubernetes.io/bound-by-controller: yes
                   pv.kubernetes.io/provisioned-by: kubernetes.io/aws-ebs
Finalizers:        [kubernetes.io/pv-protection]
StorageClass:      gp2
Status:            Bound
Claim:             mssql/mssql-data
Reclaim Policy:    Delete
Access Modes:      RWO
VolumeMode:        Filesystem
Capacity:          1Gi
Node Affinity:
  Required Terms:
    Term 0:        failure-domain.beta.kubernetes.io/zone in [ap-southeast-2c]
                   failure-domain.beta.kubernetes.io/region in [ap-southeast-2]
Message:
Source:
    Type:       AWSElasticBlockStore (a Persistent Disk resource in AWS)
    VolumeID:   aws://ap-southeast-2c/vol-087cd704ef36ad587
    FSType:     ext4
    Partition:  0
    ReadOnly:   false
Events:         <none>
```

## What if the pod restarts?
If the pod restarts, the newly scheduled pod will use the existing PVC and your data will still be there!

This may cause transaction problems in your database, but for the most part the data will still be there.

## Conclusion
Persistent volumes are cool, and they make your life running databases, but really persisting any storage on Kubernetes easier.

I hope you had fun reading, look out for more topics soon!
