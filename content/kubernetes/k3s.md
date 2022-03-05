+++
title = "k3s - small kubernetes - part 2"
date = "2022-02-21"
aliases = ["k3s"]
tags = ["kubernetes", "containers", "micro"]
categories = ["kubernetes", "software", "dev"]
[ author ]
  name = "codecowboy.io"
+++

## Intro
I recently sat down and thought to myself "I wonder how the current market of mini or micro kubernetes distributions is going". I've been using minikube for a bit and thought to myself it was about time to start using a few other distributions.

So I have been experimenting with different micro or mini distributions. 

This post is about **k0s**.

## k3s
**k3s** is an open source project that you can find here: [https://k3s.io/] (https://k3s.io/).

k3s is an open soure project and is part of the CNCF - [https://www.cncf.io/projects/k3s/] (https://www.cncf.io/projects/k3s/) 

k3s is sponsored and developed primarily by Rancher.

## Install

The installation is very easy and similarly low touch as the other distributuions I've looked at so far.

### Single Node Cluster

Single node cluster installation is really easy is you use the defaults. 

Run the command:

```Bash
curl -sfL https://get.k3s.io | sh -
```

This will download and install the latest stable version of k3s by default for your platform. As I typically use Fedora [https://getfedora.org/] (https://getfedora.org/), I was initially very impressed that my OS of choice was automatically detected, and the appropraite packages were downloaded and installed without any problems.

```Bash
[root@kube ~]# curl -sfL https://get.k3s.io | sh -
[INFO]  Finding release for channel stable
[INFO]  Using v1.22.6+k3s1 as release
[INFO]  Downloading hash https://github.com/k3s-io/k3s/releases/download/v1.22.6                               +k3s1/sha256sum-amd64.txt
[INFO]  Downloading binary https://github.com/k3s-io/k3s/releases/download/v1.22                               .6+k3s1/k3s
[INFO]  Verifying binary download
[INFO]  Installing k3s to /usr/local/bin/k3s
Fedora 35 - x86_64 - Updates                    4.7 kB/s | 4.2 kB     00:00
Fedora 35 - x86_64 - Updates                    1.3 MB/s | 3.1 MB     00:02
Fedora Modular 35 - x86_64 - Updates            4.8 kB/s | 3.9 kB     00:00
Fedora Modular 35 - x86_64 - Updates            1.4 MB/s | 2.8 MB     00:02
Rancher K3s Common (stable)                     886  B/s | 1.6 kB     00:01
Dependencies resolved.
================================================================================
 Package            Arch    Version            Repository                  Size
================================================================================
Installing:
 k3s-selinux        noarch  0.5-1.el8          rancher-k3s-common-stable   19 k
Installing dependencies:
 container-selinux  noarch  2:2.169.0-1.fc35   fedora                      50 k

Transaction Summary
================================================================================
Install  2 Packages

Total download size: 69 k
Installed size: 138 k
Downloading Packages:
(1/2): container-selinux-2.169.0-1.fc35.noarch. 442 kB/s |  50 kB     00:00
(2/2): k3s-selinux-0.5-1.el8.noarch.rpm          16 kB/s |  19 kB     00:01
--------------------------------------------------------------------------------
Total                                            34 kB/s |  69 kB     00:02
Rancher K3s Common (stable)                     3.8 kB/s | 2.4 kB     00:00
Importing GPG key 0xE257814A:
 Userid     : "Rancher (CI) <ci@rancher.com>"
 Fingerprint: C8CF F216 4551 26E9 B9C9 18BE 925E A29A E257 814A
 From       : https://rpm.rancher.io/public.key
Key imported successfully
Running transaction check
Transaction check succeeded.
Running transaction test
Transaction test succeeded.
Running transaction
  Preparing        :                                                        1/1
  Running scriptlet: container-selinux-2:2.169.0-1.fc35.noarch              1/2
  Installing       : container-selinux-2:2.169.0-1.fc35.noarch              1/2
  Running scriptlet: container-selinux-2:2.169.0-1.fc35.noarch              1/2
  Running scriptlet: k3s-selinux-0.5-1.el8.noarch                           2/2
  Installing       : k3s-selinux-0.5-1.el8.noarch                           2/2
  Running scriptlet: k3s-selinux-0.5-1.el8.noarch                           2/2
  Running scriptlet: container-selinux-2:2.169.0-1.fc35.noarch              2/2
  Running scriptlet: k3s-selinux-0.5-1.el8.noarch                           2/2
  Verifying        : container-selinux-2:2.169.0-1.fc35.noarch              1/2
  Verifying        : k3s-selinux-0.5-1.el8.noarch                           2/2

Installed:
  container-selinux-2:2.169.0-1.fc35.noarch     k3s-selinux-0.5-1.el8.noarch

Complete!
[INFO]  Skipping /usr/local/bin/kubectl symlink to k3s, command exists in PATH a                               t /root/./kubectl
[INFO]  Creating /usr/local/bin/crictl symlink to k3s
[INFO]  Creating /usr/local/bin/ctr symlink to k3s
[INFO]  Creating killall script /usr/local/bin/k3s-killall.sh
[INFO]  Creating uninstall script /usr/local/bin/k3s-uninstall.sh
[INFO]  env: Creating environment file /etc/systemd/system/k3s.service.env
[INFO]  systemd: Creating service file /etc/systemd/system/k3s.service
[INFO]  systemd: Enabling k3s unit
Created symlink /etc/systemd/system/multi-user.target.wants/k3s.service → /etc/s                               ystemd/system/k3s.service.
[INFO]  systemd: Starting k3s
```
Checking the view of what's running by default, I see a few interesting things.

```Bash
[root@kube ~]# k3s kubectl get all -A
NAMESPACE     NAME                                          READY   STATUS      RESTARTS   AGE
kube-system   pod/coredns-96cc4f57d-t7xjd                   1/1     Running     0          11m
kube-system   pod/local-path-provisioner-84bb864455-rqrk2   1/1     Running     0          11m
kube-system   pod/metrics-server-ff9dbcb6c-kmdqf            0/1     Running     0          11m
kube-system   pod/helm-install-traefik-crd--1-knz4j         0/1     Completed   0          11m
kube-system   pod/helm-install-traefik--1-2j7cd             0/1     Completed   2          11m
kube-system   pod/svclb-traefik-jm6vj                       2/2     Running     0          9m53s
kube-system   pod/traefik-55fdc6d984-l8ppl                  1/1     Running     0          9m54s

NAMESPACE     NAME                     TYPE           CLUSTER-IP      EXTERNAL-IP      PORT(S)                      AGE
default       service/kubernetes       ClusterIP      10.43.0.1       <none>           443/TCP                      12m
kube-system   service/kube-dns         ClusterIP      10.43.0.10      <none>           53/UDP,53/TCP,9153/TCP       11m
kube-system   service/metrics-server   ClusterIP      10.43.217.160   <none>           443/TCP                      11m
kube-system   service/traefik          LoadBalancer   10.43.16.118    192.168.50.200   80:32550/TCP,443:31314/TCP   9m54s

NAMESPACE     NAME                           DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
kube-system   daemonset.apps/svclb-traefik   1         1         1       1            1           <none>          9m54s

NAMESPACE     NAME                                     READY   UP-TO-DATE   AVAILABLE   AGE
kube-system   deployment.apps/coredns                  1/1     1            1           11m
kube-system   deployment.apps/local-path-provisioner   1/1     1            1           11m
kube-system   deployment.apps/traefik                  1/1     1            1           9m54s
kube-system   deployment.apps/metrics-server           0/1     1            0           11m

NAMESPACE     NAME                                                DESIRED   CURRENT   READY   AGE
kube-system   replicaset.apps/metrics-server-ff9dbcb6c            1         1         0       11m
kube-system   replicaset.apps/coredns-96cc4f57d                   1         1         1       11m
kube-system   replicaset.apps/local-path-provisioner-84bb864455   1         1         1       11m
kube-system   replicaset.apps/traefik-55fdc6d984                  1         1         1       9m54s

NAMESPACE     NAME                                 COMPLETIONS   DURATION   AGE
kube-system   job.batch/helm-install-traefik-crd   1/1           81s        11m
kube-system   job.batch/helm-install-traefik       1/1           100s       11m
```

Traefik appears to be installed by default, **and** it has an ingress enabled by default!


### Multi Node Cluster
Multi node cluster deployment is essentially the same as a single node deployment, with an additional step to install and join additional nodes to the cluster.

Install the cluster master node as per a single cluster. This is the same step as above.

```Bash
curl -sfL https://get.k3s.io | sh -
```

On the first node in your cluster, after a successful installation, check for a node token. This is used to join other nodes to the cluster.

``` Bash
# cat /var/lib/rancher/k3s/server/node-token

# K107368ba05971c9f7b89fc3c8612f31371ccb26dbd2fa0f1f78d52f7323daf9cbc::server:ac873d131ebcb77ed65dfdfd4500482b
```

Run the following command to download k3s and install it on the node. Pass the additional configuration options of:

**K3S_URL** - This is the IP or hostname of the api server. The first node that you installed.
**K3S_TOKEN** - This is the token that comes from the first node to allow other nodes to join the cluster.


```Bash
# curl -sfL https://get.k3s.io | K3S_URL=https://192.168.50.200:6443 K3S_TOKEN=K107368ba05971c9f7b89fc3c8612f31371ccb26dbd2fa0f1f78d52f7323daf9cbc::server:ac873d131ebcb77ed65dfdfd4500482b sh -
```

#### A Note on SSH

## Deployment

Again, I'll use my star wars API as an example deployment.

```Yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: svk-swapi-api
  labels:
    app: svk-swapi
spec:
  replicas: 2
  selector:
    matchLabels:
      app: svk-swapi
  template:
    metadata:
      labels:
        app: svk-swapi
    spec:
      containers:
      - image: public.ecr.aws/y6q2t0j9/demos:swapi-api
        imagePullPolicy: IfNotPresent
        name: svk-swapi
        ports:
          - containerPort: 3000
---
apiVersion: v1
kind: Service
metadata:
  name: swapi-svc
  labels:
    app: svk-swapi
spec:
  type: ClusterIP
  selector:
    app: svk-swapi
  ports:
  - name: port
    port: 3000
    targetPort: 3000
```

I create a deployment and a service for my starwars container using the manifest above. 
I can deploy this in the following way.

```Bash
kubectl --kubeconfig /etc/rancher/k3s/k3s.yaml apply -f swapi.yaml
deployment.apps/svk-swapi-api created
service/swapi-svc created
```

We can see that the pod have been deployed successfully.

```Bash
kubectl get pods
NAME                             READY   STATUS    RESTARTS   AGE
svk-swapi-api-6bbf7b44dd-hlsgt   1/1     Running   0          32s
svk-swapi-api-6bbf7b44dd-2krl5   1/1     Running   0          32s
```

Part of my deployment is to create a service. 

```Yaml
 kubectl get svc
NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
swapi-svc    ClusterIP   10.43.241.158   <none>        3000/TCP   63s
```

I can validate that the service is working on the node by using curl

```Bash
curl 10.43.241.158:3000/people/1

{
  "edited": "2014-12-20T21:17:56.891Z",
  "name": "Luke Skywalker",
  "created": "2014-12-09T13:50:51.644Z",
  "gender": "male",
  "skin_color": "fair",
  "hair_color": "blond",
  "height": "172",
  "eye_color": "blue",
  "mass": "77",
  "homeworld": 1,
  "birth_year": "19BBY",
  "image": "luke_skywalker.jpg",
  "id": 1,
  "vehicles": [
    14,
    30
  ],
  "starships": [
    12,
    22
  ],
  "films": [
    1,
    2,
    3,
    6
  ]
}
```

## Storage

## CNI

## Ingress

## Conclusion

