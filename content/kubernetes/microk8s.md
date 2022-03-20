+++
title = "microk8s - small kubernetes - part 3"
date = "2022-03-20"
aliases = ["microk8s"]
tags = ["kubernetes", "containers", "micro"]
categories = ["kubernetes", "software", "dev"]
[ author ]
  name = "codecowboy.io"
+++

## Intro
I recently sat down and thought to myself "I wonder how the current market of mini or micro kubernetes distributions is going". I've been using minikube for a bit and thought to myself it was about time to start using a few other distributions.

So I have been experimenting with different micro or mini distributions. 

This post is about **microk8s**.

## microk8s
**microk8s** is an open source project that you can find here: [https://microk8s.io/] (https://microk8s.io/).


microk8s is sponsored and developed primarily by Canonical.

## Install

The installation is very easy and similarly low touch as the other distributuions I've looked at so far.

### Single Node Cluster

Single node cluster installation is really easy if you use the defaults. 

Run the command:

```Bash
snap install microk8s --classic
```

It's a SNAP install so will work out of the box on an ubuntu distribution.

```Bash
Run configure hook of "microk8s" snap if present                                                              |
microk8s (1.23/stable) v1.23.4 from Canonical✓ installed
```

Run a check to see if **microk8s** is running.

```Bash
root@ubuntu:~# microk8s status
microk8s is running
high-availability: no
  datastore master nodes: 127.0.0.1:19001
  datastore standby nodes: none
addons:
  enabled:
    ha-cluster           # Configure high availability on the current node
  disabled:
    ambassador           # Ambassador API Gateway and Ingress
    cilium               # SDN, fast with full network policy
    dashboard            # The Kubernetes dashboard
    dashboard-ingress    # Ingress definition for Kubernetes dashboard
    dns                  # CoreDNS
    fluentd              # Elasticsearch-Fluentd-Kibana logging and monitoring
    gpu                  # Automatic enablement of Nvidia CUDA
    helm                 # Helm 2 - the package manager for Kubernetes
    helm3                # Helm 3 - Kubernetes package manager
    host-access          # Allow Pods connecting to Host services smoothly
    inaccel              # Simplifying FPGA management in Kubernetes
    ingress              # Ingress controller for external access
    istio                # Core Istio service mesh services
    jaeger               # Kubernetes Jaeger operator with its simple config
    kata                 # Kata Containers is a secure runtime with lightweight VMS
    keda                 # Kubernetes-based Event Driven Autoscaling
    knative              # The Knative framework on Kubernetes.
    kubeflow             # Kubeflow for easy ML deployments
    linkerd              # Linkerd is a service mesh for Kubernetes and other frameworks
    metallb              # Loadbalancer for your Kubernetes cluster
    metrics-server       # K8s Metrics Server for API access to service metrics
    multus               # Multus CNI enables attaching multiple network interfaces to pods
    openebs              # OpenEBS is the open-source storage solution for Kubernetes
    openfaas             # OpenFaaS serverless framework
    portainer            # Portainer UI for your Kubernetes cluster
    prometheus           # Prometheus operator for monitoring and logging
    rbac                 # Role-Based Access Control for authorisation
    registry             # Private image registry exposed on localhost:32000
    storage              # Storage class; allocates storage from host directory
    traefik              # traefik Ingress controller for external access
```

The status command allows you to install various add-ons, such a Ingress Load Balancers and Istio. This is a fantastically easy way to try out various types of software.

I'm goint to enable some of the addons before installing a second node.

```Bash
root@ubuntu:~# microk8s enable dashboard dns registry
Enabling Kubernetes Dashboard
Enabling Metrics-Server
serviceaccount/metrics-server created
clusterrole.rbac.authorization.k8s.io/system:aggregated-metrics-reader created
clusterrole.rbac.authorization.k8s.io/system:metrics-server created
rolebinding.rbac.authorization.k8s.io/metrics-server-auth-reader created
clusterrolebinding.rbac.authorization.k8s.io/metrics-server:system:auth-delegator created
clusterrolebinding.rbac.authorization.k8s.io/system:metrics-server created
service/metrics-server created
deployment.apps/metrics-server created
apiservice.apiregistration.k8s.io/v1beta1.metrics.k8s.io created
clusterrolebinding.rbac.authorization.k8s.io/microk8s-admin created
Metrics-Server is enabled
Applying manifest
serviceaccount/kubernetes-dashboard created
service/kubernetes-dashboard created
secret/kubernetes-dashboard-certs created
secret/kubernetes-dashboard-csrf created
secret/kubernetes-dashboard-key-holder created
configmap/kubernetes-dashboard-settings created
role.rbac.authorization.k8s.io/kubernetes-dashboard created
clusterrole.rbac.authorization.k8s.io/kubernetes-dashboard created
rolebinding.rbac.authorization.k8s.io/kubernetes-dashboard created
clusterrolebinding.rbac.authorization.k8s.io/kubernetes-dashboard created
deployment.apps/kubernetes-dashboard created
service/dashboard-metrics-scraper created
deployment.apps/dashboard-metrics-scraper created

If RBAC is not enabled access the dashboard using the default token retrieved with:

token=$(microk8s kubectl -n kube-system get secret | grep default-token | cut -d " " -f1)
microk8s kubectl -n kube-system describe secret $token

In an RBAC enabled setup (microk8s enable RBAC) you need to create a user with restricted
permissions as shown in:
https://github.com/kubernetes/dashboard/blob/master/docs/user/access-control/creating-sample-user.md

Enabling DNS
Applying manifest
serviceaccount/coredns created
configmap/coredns created
deployment.apps/coredns created
service/kube-dns created
clusterrole.rbac.authorization.k8s.io/coredns created
clusterrolebinding.rbac.authorization.k8s.io/coredns created
Restarting kubelet
DNS is enabled

The registry will be created with the default size of 20Gi.
You can use the "size" argument while enabling the registry, eg microk8s.enable registry:size=30Gi
Enabling default storage class
deployment.apps/hostpath-provisioner created
storageclass.storage.k8s.io/microk8s-hostpath created
serviceaccount/microk8s-hostpath created
clusterrole.rbac.authorization.k8s.io/microk8s-hostpath created
clusterrolebinding.rbac.authorization.k8s.io/microk8s-hostpath created
Storage will be available soon
Applying registry manifest
namespace/container-registry created
persistentvolumeclaim/registry-claim created
deployment.apps/registry created
service/registry created
configmap/local-registry-hosting configured
The registry is enabled
```

The command above will enable the dashboard, coredns and a registry. Technically, I don't need a registry, but it's nice to know it's there. 

When I come to the ingress portions below, I will enable various default ingress controllers.

I can validate what's running inside my cluster like this:

First I get a kubeconfig so that I can use native kubectl commands.

```Bash
root@ubuntu:~# microk8s config > kubeconfig
root@ubuntu:~# more kubeconfig
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUREekNDQWZlZ0F3SUJBZ0lVTkR0R1dUUjBqc21
OSWhUbTBRZTdmRDBYcVNrd0RRWUpLb1pJaHZjTkFRRUwKQlFBd0Z6RVZNQk1HQTFVRUF3d01NVEF1TVRVeUxqRTRNeTR4TUI0WERUSXlNRE15TU
RBeU16VTBNRm9YRFRNeQpNRE14TnpBeU16VTBNRm93RnpFVk1CTUdBMVVFQXd3TU1UQXVNVFV5TGpFNE15NHhNSUlCSWpBTkJna3Foa2lHCjl3M
EJBUUVGQUFPQ0FROEFNSUlCQ2dLQ0FRRUF1MXBOVDFhb21lSU9FbE5UcThFNndabVA2MFQ2b1JnUTlXeUsKaGlvVVJyaHQ0dlk3emIrRXFHRjRQ
MU9ta25LRTRjS3lYbk9mTTgxWS9qL21LSjh2VGt0Qy9XMWs4RE5keVdhZApUcXVZL2lNR002QnZBWkE3blJiUHovZW13ckErakoyUFJxaDFkbmF
vbFBIL01QY3N3R0dQRmErOHlLYW9kNFZVCjc2QXhBaXUvMGt0SWMvbnF6ZllKOXBaQTJVemRjK0NOMjlwcW1WV2svTDJ5OTFwU0pWckxDQ3MvST
k0U2tjY1gKNTJrTUl3M291cHlxVThwclVFSlE2U2xqU0JCQmFZYUVFMytQRWx4OEx3TDg4Rk4rdDY5Wnk1N1FyeEp4cWtmbgo1bDRSWkF6WktsT
FVicDVMY1lBYVptTjJzWlVYL0llaVB2OTFJSXpFMFFVS0NRM0Mvd0lEQVFBQm8xTXdVVEFkCkJnTlZIUTRFRmdRVWtzcktTVEJkQlVsM0N0Z3Vv
MnB4TUEvejlQZ3dId1lEVlIwakJCZ3dGb0FVa3NyS1NUQmQKQlVsM0N0Z3VvMnB4TUEvejlQZ3dEd1lEVlIwVEFRSC9CQVV3QXdFQi96QU5CZ2t
xaGtpRzl3MEJBUXNGQUFPQwpBUUVBV1JCSFJFYzdOUVcyb0FERk5xeUtla2VSUlAzcEtsNGd1L3k3czVTRFhGSVZZM0pKZnVuc2tyMEpQeHBuCn
h2QnpXd3lQL3lZbml4VzQ1NVFHK0ZkWTRHWm5aOWMzV25oSWtJRTJYbnJ6dlQ2L0l3anpWK2N3M1BDSGs1RnMKWmQvL2sxU3B5L2R3dDZ1VTBDV
jlLdytRM1J3ZjhLRmVHTlBKNGM0b0JtOTBpRmQ4cFdMUWNhT0NsbTlzV1dDZApTd3BvRmJZRXk0bjM5V1BMcWJZbU9ZRlZUQ2dUUEpFVFk1aFA2
eDMvNFpjN1BCL2RGQUNQWEUzMDRmYVJjKzZpCmlRMjZzcDNkM1JRSjlzTnNFOGpKQjdHOURSUWM4VXJKekc5Y0cxUW9FRFBXNkNrcnJJK1NLMEt
GWDF0VFhXS3AKVGw4UmhxUG5SbjlUb1ZhMlFPa0FBbCtKMmc9PQotLS0tLUVORCBDRVJUSUZJQ0FURS0tLS0tCg==
    server: https://192.168.50.110:16443
  name: microk8s-cluster
contexts:
- context:
    cluster: microk8s-cluster
    user: admin
  name: microk8s
current-context: microk8s
kind: Config
preferences: {}
users:
- name: admin
  user:
    token: cjJtaUZwaU1rMUVtQW9lVjZhTEZFd1dlcFV1bFR3Q0MySzNUNERqQlg3QT0K
```

Now I can simply export my KUBECONFIG environment variable and kubectl should work as normal.

I can see that I have nodes and pods.

```Bash
root@ubuntu:~# kubectl get nodes
NAME     STATUS   ROLES    AGE     VERSION
ubuntu   Ready    <none>   9m53s   v1.23.4-2+98fc2022f3ad3e
```

```Bash
root@ubuntu:~# kubectl get pods -A
NAMESPACE            NAME                                         READY   STATUS    RESTARTS   AGE
kube-system          calico-node-7cm7m                            1/1     Running   0          10m
kube-system          calico-kube-controllers-758bff5fc6-lw582     1/1     Running   0          10m
kube-system          metrics-server-679c5f986d-28kln              1/1     Running   0          5m44s
kube-system          coredns-64c6478b6c-7rp6g                     1/1     Running   0          4m12s
kube-system          kubernetes-dashboard-585bdb5648-mj78b        1/1     Running   0          4m12s
kube-system          dashboard-metrics-scraper-69d9497b54-mfbrb   1/1     Running   0          4m12s
kube-system          hostpath-provisioner-7764447d7c-qdxr7        1/1     Running   0          4m12s
container-registry   registry-5f697bb7df-pk2kr                    1/1     Running   0          4m12s
```

The pods include the dashboard, coredns and a registry - all of the addons I installed earlier.

### Multi Node Cluster
Adding additional nodes is really easy. 
It's a similar process to other micro kubernetes distributions in that you need to have a key to add the second node. 

On the first node you installed run the following command:

```Bash
root@ubuntu:~# microk8s add-node
From the node you wish to join to this cluster, run the following:
microk8s join 192.168.50.110:25000/3728d3a71c3a81c8da4ab4783cfee2ff/3524ecb2590c

Use the '--worker' flag to join a node as a worker not running the control plane, eg:
microk8s join 192.168.50.110:25000/3728d3a71c3a81c8da4ab4783cfee2ff/3524ecb2590c --worker

If the node you are adding is not reachable through the default interface you can use one of the following:
microk8s join 192.168.50.110:25000/3728d3a71c3a81c8da4ab4783cfee2ff/3524ecb2590c
```

This command outputs the exact command that you need to use on the worker node to join to the first node you installed. This is quite nice and handy and means that you don't actually need to mess around with files and so on.

On the new **worker** node simply run the appropriate command. 
In my case the second node will be a worker only.

```Bash
root@worker:~# snap install microk8s --classic
microk8s (1.23/stable) v1.23.4 from Canonical✓ installed

root@worker:~# microk8s join 192.168.50.110:25000/3728d3a71c3a81c8da4ab4783cfee2ff/3524ecb2590c --worker
Contacting cluster at 192.168.50.110

The node has joined the cluster and will appear in the nodes list in a few seconds.

Currently this worker node is configured with the following kubernetes API server endpoints:
    - 192.168.50.110 and port 16443, this is the cluster node contacted during the join operation.

If the above endpoints are incorrect, incomplete or if the API servers are behind a loadbalancer please update
/var/snap/microk8s/current/args/traefik/provider.yaml
```

Once this completes you should have an api server and a single worker node.


We can validate this by checking the nodes from the API server

```Bash
root@ubuntu:~# kubectl get nodes
NAME     STATUS   ROLES    AGE   VERSION
worker   Ready    <none>   60s   v1.23.4-2+98fc2022f3ad3e
ubuntu   Ready    <none>   20m   v1.23.4-2+98fc2022f3ad3e
```

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

I deploy this and check the deployment and service:

```Bash
root@ubuntu:~# kubectl apply -f swapi-1.yaml
deployment.apps/svk-swapi-api created
service/swapi-svc created

root@ubuntu:~# kubectl get pods
NAME                             READY   STATUS    RESTARTS   AGE
svk-swapi-api-74dc79dcc4-5k9q9   1/1     Running   0          103s
svk-swapi-api-74dc79dcc4-r6tc6   1/1     Running   0          103s
```

I can see that there is a service running as well and that this corresponds to the service in my manifest. 

```Bash
root@ubuntu:~# kubectl get svc
NAME         TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
kubernetes   ClusterIP   10.152.183.1     <none>        443/TCP    34m
swapi-svc    ClusterIP   10.152.183.131   <none>        3000/TCP   2m18s
```

I can test the service and should get the familiar first result from my API

```Bash
root@ubuntu:~# curl 10.152.183.131:3000/people/1
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

There is a default storage class provisioned by default in **microk8s**. This is the hostpath storage class. 

```Bash
root@ubuntu:~# kubectl get sc
NAME                          PROVISIONER            RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
microk8s-hostpath (default)   microk8s.io/hostpath   Delete          Immediate           false                  36m
```

### OpenEBS

Using **microk8s'** inbuilt add-ons, it is possible to enable other storage provisioners. One that is included as an add-on is openEBS.

You will need to ensure that iscsid is installed on all nodes within the cluster before enabling openebs.

```Bash
sudo systemctl enable iscsid
```

Now you can run the command to enable openebs

```Bash
microk8s enable openebs
```

This will give you the following output:

```Bash
root@ubuntu:~# microk8s enable openebs
Addon dns is already enabled.
Enabling Helm 3
Fetching helm version v3.5.0.
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100 11.7M  100 11.7M    0     0  3272k      0  0:00:03  0:00:03 --:--:-- 3271k
Helm 3 is enabled
WARNING: Kubernetes configuration file is group-readable. This is insecure. Location: /var/snap/microk8s/3021/credentials/client.config
"openebs" has been added to your repositories
WARNING: Kubernetes configuration file is group-readable. This is insecure. Location: /var/snap/microk8s/3021/credentials/client.config
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "openebs" chart repository
Update Complete. ⎈Happy Helming!⎈
WARNING: Kubernetes configuration file is group-readable. This is insecure. Location: /var/snap/microk8s/3021/credentials/client.config
NAME: openebs
LAST DEPLOYED: Sun Mar 20 03:13:51 2022
NAMESPACE: openebs
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
Successfully installed OpenEBS.

Check the status by running: kubectl get pods -n openebs

The default values will install NDM and enable OpenEBS hostpath and device
storage engines along with their default StorageClasses. Use `kubectl get sc`
to see the list of installed OpenEBS StorageClasses.

**Note**: If you are upgrading from the older helm chart that was using cStor
and Jiva (non-csi) volumes, you will have to run the following command to include
the older provisioners:

helm upgrade openebs openebs/openebs \
        --namespace openebs \
        --set legacy.enabled=true \
        --reuse-values

For other engines, you will need to perform a few more additional steps to
enable the engine, configure the engines (e.g. creating pools) and create
StorageClasses.

For example, cStor can be enabled using commands like:

helm upgrade openebs openebs/openebs \
        --namespace openebs \
        --set cstor.enabled=true \
        --reuse-values

For more information,
- view the online documentation at https://openebs.io/ or
- connect with an active community on Kubernetes slack #openebs channel.
OpenEBS is installed


-----------------------

When using OpenEBS with a single node MicroK8s, it is recommended to use the openebs-hostpath StorageClass
An example of creating a PersistentVolumeClaim utilizing the openebs-hostpath StorageClass


kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: local-hostpath-pvc
spec:
  storageClassName: openebs-hostpath
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5G



-----------------------

If you are planning to use OpenEBS with multi nodes, you can use the openebs-jiva-csi-default StorageClass.
An example of creating a PersistentVolumeClaim utilizing the openebs-jiva-csi-default StorageClass


kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: jiva-volume-claim
spec:
  storageClassName: openebs-jiva-csi-default
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5G
```

I really like the way that this distribution gives helpful configuration hints when you install add-ons.

If I check storage classes now, I can see the following:

```Bash
root@ubuntu:~# kubectl get sc
NAME                          PROVISIONER            RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
microk8s-hostpath (default)   microk8s.io/hostpath   Delete          Immediate              false                  34m
openebs-device                openebs.io/local       Delete          WaitForFirstConsumer   false                  2m50s
openebs-hostpath              openebs.io/local       Delete          WaitForFirstConsumer   false                  2m50s
openebs-jiva-csi-default      jiva.csi.openebs.io    Delete          Immediate              true                   2m50s
```

## CNI

The default CNI in **microk8s** is calico. This is tried and tested, and just wokrs. Using the add-on architecture, there are other network providers that can be installed and swapped out easily.

Included are: **multus** as a CNI, and cillium as a fuill SDN - depending on what your needs are. 

Again, each of these can be turned on or off by the **microk8s enable** or **microk8s disable** command.

## Ingress

This is where things get fun. There are two ingress controllers provided by **microk8s** out of the box. NGINX and Traefik.

### NGINX Ingress

To install the NGINX ingress (I still bleed green a little) you can use the following steps:

```Bash
root@ubuntu:~# microk8s enable ingress
Enabling Ingress
ingressclass.networking.k8s.io/public created
namespace/ingress created
serviceaccount/nginx-ingress-microk8s-serviceaccount created
clusterrole.rbac.authorization.k8s.io/nginx-ingress-microk8s-clusterrole created
role.rbac.authorization.k8s.io/nginx-ingress-microk8s-role created
clusterrolebinding.rbac.authorization.k8s.io/nginx-ingress-microk8s created
rolebinding.rbac.authorization.k8s.io/nginx-ingress-microk8s created
configmap/nginx-load-balancer-microk8s-conf created
configmap/nginx-ingress-tcp-microk8s-conf created
configmap/nginx-ingress-udp-microk8s-conf created
daemonset.apps/nginx-ingress-microk8s-controller created
Ingress is enabled
```

This gives you the default ingress controller of NGINX.

First I can validate that the installation worked in the following way:

```Bash
root@ubuntu:~# kubectl get pods -n ingress
NAME                                      READY   STATUS    RESTARTS   AGE
nginx-ingress-microk8s-controller-sbjtn   1/1     Running   0          76s
nginx-ingress-microk8s-controller-9m6zw   1/1     Running   0          76s
root@ubuntu:~#
```

A namespace named "ingress" is created.

If I redeploy my manifest, and add in an ingress portion up the top, you can see that it defines a standard NGINX style ingress with no fancy annotations.

The only difference to my previous deployment is the addition of ingress, and the change of the port on the container and ingress to port 80.

```Yaml
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: swapi-ingress
spec:
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: swapi-svc
            port:
              number: 80
---
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
  selector:
    app: svk-swapi
  ports:
  - name: port
    port: 80
    targetPort: 3000
```

Let's apply this:

```Bash
root@ubuntu:~# kubectl apply -f swapi-3.yaml
ingress.networking.k8s.io/swapi-ingress created
deployment.apps/svk-swapi-api unchanged
service/swapi-svc configured

root@ubuntu:~# kubectl get svc
NAME         TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.152.183.1     <none>        443/TCP   54m
swapi-svc    ClusterIP   10.152.183.131   <none>        80/TCP    22m

root@ubuntu:~# kubectl get ingress
NAME            CLASS    HOSTS   ADDRESS   PORTS   AGE
swapi-ingress   public   *                 80      11s
```

We can see that the Ingress is created on port 80 as is the service.

I can connect to the ingress on the hosts public IP address of my worker node like this:

```Bash
root@ubuntu:~# curl worker:80/people/1
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

NGINX Ingress works out of the box!

I can just as easily disable NGINX Ingress like this:

```Bash
root@ubuntu:~# microk8s disable ingress
Disabling Ingress
ingressclass.networking.k8s.io "public" deleted
namespace "ingress" deleted
serviceaccount "nginx-ingress-microk8s-serviceaccount" deleted
clusterrole.rbac.authorization.k8s.io "nginx-ingress-microk8s-clusterrole" deleted
role.rbac.authorization.k8s.io "nginx-ingress-microk8s-role" deleted
clusterrolebinding.rbac.authorization.k8s.io "nginx-ingress-microk8s" deleted
rolebinding.rbac.authorization.k8s.io "nginx-ingress-microk8s" deleted
configmap "nginx-load-balancer-microk8s-conf" deleted
configmap "nginx-ingress-tcp-microk8s-conf" deleted
configmap "nginx-ingress-udp-microk8s-conf" deleted
daemonset.apps "nginx-ingress-microk8s-controller" deleted
Ingress is disabled
```

### Traefik Ingress

**microk8s** also comes with Traefik for Ingress. This is super cool to have two ingress types that I can swith on and off and just use. 

I enable it like this:

```Bash
root@ubuntu:~# microk8s enable traefik
Enabling traefik ingress controller on port 8080
serviceaccount/traefik-ingress-controller created
daemonset.apps/traefik-ingress-controller created
service/traefik-ingress-service created
clusterrole.rbac.authorization.k8s.io/traefik-ingress-controller created
clusterrolebinding.rbac.authorization.k8s.io/traefik-ingress-controller created
service/traefik-web-ui created
ingress.networking.k8s.io/traefik-web-ui created
traefik ingress controller has been installed on port 8080
```

Again, I can re-deploy my star wars API with an additional annotation to denote that it is a traefik ingress. 

Note that the ingress definition has an additional annotation that tells the ingress that it is a class of **traefik**. Other than the annotation it is the same as the NGINX ingress above.

```Yaml
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: swapi-ingress
  annotations:
    kubernetes.io/ingress.class: "traefik"
spec:
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: swapi-svc
            port:
              number: 80
---
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
  selector:
    app: svk-swapi
  ports:
  - name: port
    port: 80
    targetPort: 3000
```

Let's deploy this ingress and see what happens.

```Bash
root@ubuntu:~# kubectl apply -f swapi-2.yaml
ingress.networking.k8s.io/swapi-ingress created
deployment.apps/svk-swapi-api unchanged
service/swapi-svc configured

root@ubuntu:~# kubectl get svc
NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.152.183.1    <none>        443/TCP   63m
swapi-svc    ClusterIP   10.152.183.91   <none>        80/TCP    5m4s

root@ubuntu:~# kubectl get ingress
NAME            CLASS    HOSTS   ADDRESS     PORTS   AGE
swapi-ingress   <none>   *       127.0.0.1   80      13s
```
Similarly, I can test the ingress again, and should see the same person as in all of my other examples.

```Bash
root@ubuntu:~# curl worker:8080/people/1
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

The observant among you may note that traefik runs on a different port, port 8080, and that I had to use port 8080 in my curl command to get to my service.

If I describe the traefik pods, I can see that they have been started using port 8080.

```Bash
root@ubuntu:~# kubectl describe pod traefik-ingress-controller-fgql2 -n traefik
Name:         traefik-ingress-controller-fgql2
Namespace:    traefik
Priority:     0
Node:         ubuntu/192.168.50.110
Start Time:   Sun, 20 Mar 2022 03:36:18 +0000
Labels:       controller-revision-hash=597444df48
              k8s-app=traefik-ingress-lb
              name=traefik-ingress-lb
              pod-template-generation=1
Annotations:  <none>
Status:       Running
IP:           192.168.50.110
IPs:
  IP:           192.168.50.110
Controlled By:  DaemonSet/traefik-ingress-controller
Containers:
  traefik-ingress-lb:
    Container ID:  containerd://193816ebac3af45f3bd568d551221f954a9008b2df400c291d39d29a76454e54
    Image:         traefik:2.5
    Image ID:      docker.io/library/traefik@sha256:7d5a6ae66572b27afe1059ce893b59acb28de70f4ce298385e153d0a398ce7f8
    Port:          8080/TCP
    Host Port:     8080/TCP
    Args:
      --providers.kubernetesingress=true
      --providers.kubernetesingress.ingressendpoint.ip=127.0.0.1
      --log=true
      --log.level=INFO
      --accesslog=true
      --accesslog.filepath=/dev/stdout
      --accesslog.format=json
      --entrypoints.web.address=:8080
      --entrypoints.websecure.address=:8443
    State:          Running
      Started:      Sun, 20 Mar 2022 03:36:33 +0000
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-8w4bl (ro)
Conditions:
  Type              Status
  Initialized       True
  Ready             True
  ContainersReady   True
  PodScheduled      True
Volumes:
  kube-api-access-8w4bl:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    ConfigMapOptional:       <nil>
    DownwardAPI:             true
QoS Class:                   BestEffort
Node-Selectors:              <none>
Tolerations:                 :NoSchedule op=Exists
                             node.kubernetes.io/disk-pressure:NoSchedule op=Exists
                             node.kubernetes.io/memory-pressure:NoSchedule op=Exists
                             node.kubernetes.io/network-unavailable:NoSchedule op=Exists
                             node.kubernetes.io/not-ready:NoExecute op=Exists
                             node.kubernetes.io/pid-pressure:NoSchedule op=Exists
                             node.kubernetes.io/unreachable:NoExecute op=Exists
                             node.kubernetes.io/unschedulable:NoSchedule op=Exists
Events:                      <none>
```

There are two arguments that have been passed to the containers at startup that we can see: 

```Bash
      --entrypoints.web.address=:8080
      --entrypoints.websecure.address=:8443
```

This is quite smart because it means that you can run **both** traefik and NGINX as ingress controllers at the same time if you want. Doing this by default is smart and makes life easy.

## Conclusion
**microk8s** is a good solid distribution that works well. I really like the fact that it has sensible defaults and is easy to install.

The installation again is different to **k0s** and **k3s** but is easy enough. 

One minor gripe is that I had to install ubuntu to get everything to work seamlessly. My usual distribution is Fedora, and on Fedora 35 I hit a problem with a few pods not starting correctly. I didn't bother to troubleshoot it and moved to ubuntu to see the difference. If you're an ubuntu user it probably wont bother you at all.

I would say that the add-on architecture is solid and a neat way of making the distribution user friendly. Using snap is also something lowers the barrier of entry - although again, additional steps are needed on Fedora 35. 

The add-on architecture and using HELM to install addons is super slick and very nice. The model that **microk8s** uses removes the need to manually handle helm templates and install helm yourself. I cannot overstate how good this is.

The addition of sensible defaults is also quite nice. Multiple Ingress controllers that can both be run side by side by default without further intervention is very nice. 

All in all, this is a good distribution for getting up and running quickly, and a super good distribution if you want to switch components out easily to test different scenarios.

...and I didn't even touch on the fact that Istio is bundled as an add-on as well :)
