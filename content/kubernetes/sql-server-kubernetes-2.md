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

So in my last post I showed how you could create databases on Kubernetes. There are many reasons to do this. Equally, there are reasons not to do this, but for highly distributed deployments it does make sense.

This post is going to focus on the storage components of running a database on Kubernetes.

## Why do I need persistent storage

Persistent storage as the name implies allows you to store your data between container restarts. This is important where data is stored in a database. You actually want your data to be persisted. 

## How do I do this in kubernetes?

Kubernetes has two concepts that allow you to persist data that we are going to use. The first is a persistent volume claim, the second is a persistent volume mount

## Persistent Volume Claim

In Kubernetes a persistent volume claim is a way of "grabbing some storage" if you're a user. The Kubernetes documentation describes this much more eloquently as "a request for storage from a user" and "the persistent volume API abstracts the details of storage from the user".

This is documented [here](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)

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

## Persistent Storage Volumes


## Inside the container

## How does it work?

## What if the pod restarts?

## Conclustion

ZZ
