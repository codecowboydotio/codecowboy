+++
title = "k0s - small kubernetes - part 1"
date = "2022-02-10"
aliases = ["k0s"]
tags = ["kubernetes", "containers", "micro"]
categories = ["kubernetes", "software", "dev"]
[ author ]
  name = "codecowboy.io"
+++

## Intro
I recently sat down and thought to myself "I wonder how the current market of mini or micro kubernetes distributions is going". I've been using minikube for a bit and thought to myself it was about time to start using a few other distributions.

So I have been experimenting with different micro or mini distributions. 

This post is about **k0s**.

## k0s
**k0s** is an open source project that you can find here: [https://k0sproject.io/](https://k0sproject.io/)

**k0s** looks like it's sponsored by Mirantis. They offer commercial support and so on if you want it too. In and of itself, **k0s** is a free and open source project. You can use it and it works without paying anyone :)

## Install
Installation was relatively easy for the most part. It's possible to install both a single node cluster and a multi node cluster. I'll step through both of these.

### Single Node Cluster
A single node cluster is perhaps the easiest to install and the fastest way to get up and running.

```Bash
curl -sSLf https://get.k0s.sh | sudo sh
```
This will install the k0s binary in **/usr/local/bin/**.

```Bash
sudo k0s install controller --single
```
Next install the cluster as a single node cluster.

You can check the status of your freshly installed cluster using the k0s command.

```Bash
[root@kube ~]# k0s status
Version: v1.23.3+k0s.0
Process ID: 1313
Role: controller
Workloads: true
SingleNode: true
```

It's that simple - from this point on, you have a single node cluster that can be used as normal.



### Multi Node Cluster
To install a multi node cluster, it's best to use a second binary called **k0sctl**. This can be downloaded and installed in the following way.

```Bash
curl -L https://github.com/k0sproject/k0sctl/releases/download/v0.13.0-beta.1/k0sctl-linux-x64 -o k0sctl
```

Once you have the **k0sctl** binary installed, you can then use it to create a default cluster bootstrap yaml file.

```Bash
k0sctl init > k0sctl.yaml
```

This will create a file called k0sctl.yaml that has the following contents. The file below has IP addresses that are relevant to my setup. Each node must be reachable via passwordless ssh from the node where **k0sctl** is installed. See the note below for more on this.

```Yaml
apiVersion: k0sctl.k0sproject.io/v1beta1
kind: Cluster
metadata:
  name: k0s-cluster
spec:
  hosts:
  - ssh:
      address: 10.1.1.160
      user: root
      port: 22
      keyPath: /root/.ssh/id_rsa
    role: controller
  - ssh:
      address: 10.1.1.170
      user: root
      port: 22
      keyPath: /root/.ssh/id_rsa
    role: worker
  k0s:
    version: 1.23.3+k0s.0
    dynamicConfig: false
```

If your SSH is configured correctly to all of your nodes, you should be able to run the following command:

```Bash
k0sctl apply --config ./k0sctl.yaml
```

This will perform a multi cluster installation on the two nodes specified. The output will look something like this:

```Bash

⠀⣿⣿⡇⠀⠀⢀⣴⣾⣿⠟⠁⢸⣿⣿⣿⣿⣿⣿⣿⡿⠛⠁⠀⢸⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⠀█████████ █████████ ███
⠀⣿⣿⡇⣠⣶⣿⡿⠋⠀⠀⠀⢸⣿⡇⠀⠀⠀⣠⠀⠀⢀⣠⡆⢸⣿⣿⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀███          ███    ███
⠀⣿⣿⣿⣿⣟⠋⠀⠀⠀⠀⠀⢸⣿⡇⠀⢰⣾⣿⠀⠀⣿⣿⡇⢸⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⠀███          ███    ███
⠀⣿⣿⡏⠻⣿⣷⣤⡀⠀⠀⠀⠸⠛⠁⠀⠸⠋⠁⠀⠀⣿⣿⡇⠈⠉⠉⠉⠉⠉⠉⠉⠉⢹⣿⣿⠀███          ███    ███
⠀⣿⣿⡇⠀⠀⠙⢿⣿⣦⣀⠀⠀⠀⣠⣶⣶⣶⣶⣶⣶⣿⣿⡇⢰⣶⣶⣶⣶⣶⣶⣶⣶⣾⣿⣿⠀█████████    ███    ██████████

k0sctl v0.13.0-beta.1 Copyright 2021, k0sctl authors.
Anonymized telemetry of usage will be sent to the authors.
By continuing to use k0sctl you agree to these terms:
https://k0sproject.io/licenses/eula
INFO ==> Running phase: Connect to hosts
INFO [ssh] 10.1.1.160:22: connected
INFO [ssh] 10.1.1.170:22: connected
INFO ==> Running phase: Detect host operating systems
INFO [ssh] 10.1.1.160:22: is running Fedora Linux 35 (Server Edition)
INFO [ssh] 10.1.1.170:22: is running Fedora Linux 35 (Server Edition)
INFO ==> Running phase: Prepare hosts
INFO ==> Running phase: Gather host facts
INFO [ssh] 10.1.1.160:22: using kube as hostname
INFO [ssh] 10.1.1.170:22: using kube-worker as hostname
INFO [ssh] 10.1.1.160:22: discovered ens224 as private interface
INFO [ssh] 10.1.1.170:22: discovered ens224 as private interface
INFO [ssh] 10.1.1.160:22: discovered 192.168.141.131 as private address
INFO [ssh] 10.1.1.170:22: discovered 192.168.141.132 as private address
INFO ==> Running phase: Validate hosts
INFO ==> Running phase: Gather k0s facts
INFO ==> Running phase: Validate facts
INFO ==> Running phase: Download k0s on hosts
INFO [ssh] 10.1.1.170:22: downloading k0s 1.23.3+k0s.0
INFO ==> Running phase: Configure k0s
WARN [ssh] 10.1.1.160:22: generating default configuration
INFO [ssh] 10.1.1.160:22: validating configuration
INFO [ssh] 10.1.1.160:22: configuration was changed
INFO ==> Running phase: Initialize the k0s cluster
INFO [ssh] 10.1.1.160:22: installing k0s controller
INFO [ssh] 10.1.1.160:22: waiting for the k0s service to start
INFO [ssh] 10.1.1.160:22: waiting for kubernetes api to respond
INFO ==> Running phase: Install workers
INFO [ssh] 10.1.1.170:22: validating api connection to https://192.168.141.131:6443
INFO [ssh] 10.1.1.160:22: generating token
INFO [ssh] 10.1.1.170:22: writing join token
INFO [ssh] 10.1.1.170:22: installing k0s worker
INFO [ssh] 10.1.1.170:22: starting service
INFO [ssh] 10.1.1.170:22: waiting for node to become ready
INFO ==> Running phase: Disconnect from hosts
INFO ==> Finished in 2m9s
INFO k0s cluster version 1.23.3+k0s.0 is now installed
INFO Tip: To access the cluster you can now fetch the admin kubeconfig using:
INFO      k0sctl kubeconfig
```

At this point there is a multi cluster kubernetes installation that's running and I can use as normal.
I can even run **k0sctl kubeconfig** to get my kubeconfig file and simply use that to run commands against the cluster.

```Bash
**********************************************
kubectl --kubeconfig ./k0s_kube_config get pods
```



#### A Note on SSH
I found initially I had a few weird ssh problems. I wont go into them in a lot of detail suffice to say that running the ssh-agent and adding keys was enough to make the multi cluster installation work okay.

```Bash
[root@kube ~]# eval $(ssh-agent)
Agent pid 1086
[root@kube ~]# ssh-add
Identity added: /root/.ssh/id_rsa (root@kube)
Identity added: /root/.ssh/id_ecdsa (root@kube)
```

The other odd thing was that in order for the installation to work at all I had to create both an RSA and an ECDSA key. I haven't delved into this in more detail yet, but will look over the weekend.

```Bash
ssh-keygen -t ecdsa
```

Add the keys as above, and everything should install without a problem.

## Deployment
This is where things get funky. I was very impressed to find that **k0s** has a deployer!

[https://docs.k0sproject.io/v1.23.3+k0s.0/manifests/#overview](https://docs.k0sproject.io/v1.23.3+k0s.0/manifests/#overview)

If you create a subdirectory under **/var/lib/k0s/manifests** on a controller node, and place manifests within it, these will automatically be deployed to the cluster. Pruning works as well, so if you remoe a directory, the manifest deployer will update the cluster state. 

This is a very handy way of deploying a consistent configuration across multiple nodes, or if you are stamping out multiple edge nodes that are not necessarily part of the same cluster - think automating this.

An example is below:

```Bash
apiVersion: v1
kind: Namespace
metadata:
  name: svk
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: svk-swapi-api
  namespace: svk
  labels:
    app: svk-swapi-api
spec:
  replicas: 2
  selector:
    matchLabels:
      app: svk-swapi-api
  template:
    metadata:
      labels:
        app: svk-swapi-api
    spec:
      containers:
      - image: public.ecr.aws/y6q2t0j9/demos:swapi-api
        imagePullPolicy: IfNotPresent
        name: svk-swapi-api
        ports:
          - containerPort: 3000
---
apiVersion: v1
kind: Service
metadata:
  name: swapi-svc
  namespace: svk
  labels:
    app: svk-swapi-api
spec:
  type: ClusterIP
  selector:
    app: svk-swapi-api
  ports:
  - name: port
    port: 3000
    targetPort: 3000
```

All I need to do is place the manifest above in the appropriate directory **/var/lib/k0s/manifests** and the namespace, service and pods will be created for me.

```Bash
[root@kube svk]# k0s kubectl get pods -n svk
NAMESPACE     NAME                              READY   STATUS    RESTARTS   AGE
svk           svk-swapi-api-5bb8b884d4-mkb5h    1/1     Running   0          2m36s
svk           svk-swapi-api-5bb8b884d4-tlzlh    1/1     Running   0          2m36s

[root@kube svk]# kubectl get namespace 
NAME              STATUS        AGE
default           Active        18m
kube-node-lease   Active        18m
kube-public       Active        18m
kube-system       Active        18m
svk               Active        2m36s

[root@kube svk]# k0s kubectl get svc -A
NAMESPACE     NAME             TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                  AGE
default       kubernetes       ClusterIP   10.96.0.1       <none>        443/TCP                  18m
kube-system   kube-dns         ClusterIP   10.96.0.10      <none>        53/UDP,53/TCP,9153/TCP   17m
kube-system   metrics-server   ClusterIP   10.101.198.75   <none>        443/TCP                  17m
svk           swapi-svc        ClusterIP   10.110.74.132   <none>        3000/TCP                 2m36s
```

There is also a HELM deployer as well. 

## Storage
The most impressive thing for me was that k0s has a storage provider built in by default. I Could easily create PV's and PVC's without a problem.

```Bash
[root@kube svk]# k0s kubectl get storageclass
NAME               PROVISIONER        RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
openebs-device     openebs.io/local   Delete          WaitForFirstConsumer   false                  8m39s
openebs-hostpath   openebs.io/local   Delete          WaitForFirstConsumer   false                  8m39s
```

This requires additional configuration from the default to enable, but it's two options in a yaml configuration file and a restart. Very simple.


## CNI
The default CNI is kube-router. This can be a bit limiting, so Calico is also supported. In addition to Calico, you can deploy your own CNI via the configuration file. This makes k0s quite flexible for testing and development purposes.

## Ingress


## Conclusion
I was very pleasantly surprised by **k0s** it was easy to deploy and get up and running in either a single node cluster or a multi node cluster. The documentation was very good, and **it just worked**. This is perhaps the most important part for me. As a developer, I want to have the ability to get up and running super fast, without a lot of fuss to be able to try out different configurations or deploy applications.
