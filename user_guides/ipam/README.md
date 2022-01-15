# F5 IPAM Controller User Guide - Updated for CIS 2.7

The F5 IPAM Controller is a Docker container that allocates IP addresses from an static list or Infoblox based on hostname. The F5 IPAM Controller watches CRD resources and consumes the hostname within each resource.

This diagram demonstrates the IPAM solution

![diagram](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/ipam/diagrams/2021-04-26_14-13-25.png)

Demo on YouTube [video](https://www.youtube.com/watch?v=s9iUZoRqYQs)

## Prerequisites

* Recommend AS3 version 3.30 [repo](https://github.com/F5Networks/f5-appsvcs-extension/releases/tag/v3.30.0)
* CIS 2.7 [repo](https://github.com/F5Networks/k8s-bigip-ctlr/releases/tag/v2.7.0)
* F5 IPAM Controller [repo](https://github.com/F5Networks/f5-ipam-controller/releases/tag/v0.1.6)
* CloudDocs [documentation](https://clouddocs.f5.com/containers/latest/userguide/ipam/)

## IPAM Configurable Arguments

Easy to use arguments when using the F5 IPAM controller

* Defining the IPAM label in the virtualserver CRD which maps to the IP-Range. In my example I am using the following IP range

  - ip-range='{"Test":"10.192.75.113-10.192.75.116","Production":"10.192.125.30-10.192.125.50"}'

F5 IPAM will allocate a IP address from the ip-range based on the **ipamLabel** and **hostname**

## Step 1: Configure CIS

Add the parameter --ipam=true in the CIS deployment to provide the integration with CIS and IPAM

* --ipam=true

```
args: 
  - "--bigip-username=$(BIGIP_USERNAME)"
  - "--bigip-password=$(BIGIP_PASSWORD)"
  - "--bigip-url=192.168.200.60"
  - "--bigip-partition=k8s"
  - "--namespace=default"
  - "--pool-member-type=cluster"
  - "--flannel-name=fl-vxlan"
  - "--insecure=true"
  - "--custom-resource-mode=true"
  - "--as3-validation=true"
  - "--ipam=true"
```

Deploy CIS and latest ClusterRole

```
kubectl create secret generic bigip-login -n kube-system --from-literal=username=admin --from-literal=password=<secret>
kubectl create -f bigip-ctlr-clusterrole.yaml
kubectl create -f f5-bigip-ctlr-deployment.yaml
kubectl create -f f5-bigip-node.yaml
```
**note** f5-bigip-node is for Flannel CNI


cis-deployment [repo](https://github.com/mdditt2000/k8s-bigip-ctlr/tree/main/user_guides/ipam/cis-deployment)

## Step 2: Configure F5 IPAM Controller

Use the following deployment options:

* --orchestration=kubernetes

The orchestration parameter holds the orchestration environment i.e. Kubernetes

* --ip-range='{"Test":"10.192.75.113-10.192.75.116","Production":"10.192.125.30-10.192.125.50"}'

ip-range parameter holds the IP address ranges and from this range, it creates a pool of IP address range which gets allocated to the corresponding hostname in the virtual server CRD.

* --log-level=debug

```
- args:
    - --orchestration=kubernetes
    - --ip-range='{"Test":"10.192.75.113-10.192.75.116","Production":"10.192.125.30-10.192.125.50"}'
    - --log-level=DEBUG
```

Deploy IPAM RBAC, PersistentVolume and F5 IPAM Controller

```
kubectl create -f f5-ipam-rbac.yaml
kubectl create -f f5-ipam-persitentvolume.yaml
kubectl create -f f5-ipam-deployment.yaml
```
ipam-deployment [repo](https://github.com/mdditt2000/k8s-bigip-ctlr/tree/main/user_guides/ipam/ipam-deployment)

Validate both CIS and IPAM deployments

```
❯ kubectl get pods --all-namespaces -owide |grep -i 'cis\|ipam\|dep'
kube-system   f5-ipam-controller-95dd8fc9-qtvhg                          1/1     Running   0          75m    10.244.1.40      k8s-1-19-node1.example.com    <none>           <none>
kube-system   k8s-bigip-ctlr-deployment-6469bfdd94-h2z6x                 1/1     Running   0          51m    10.244.2.157     k8s-1-19-node2.example.com    <none>           <none>
```

## Logging output from IPAM Controller

```
❯ kubectl logs -f deploy/f5-ipam-controller -n kube-system
2022/01/14 23:18:12 [INFO] [INIT] Starting: F5 IPAM Controller - Version: 0.1.5, BuildInfo: azure-1035-1bb5b0bc70546b7546ad2b1f42405b9aa867de2e
2022/01/14 23:18:12 [DEBUG] Creating IPAM Kubernetes Client
2022/01/14 23:18:12 [DEBUG] [ipam] Creating Informers for Namespace kube-system
2022/01/14 23:18:12 [DEBUG] Created New IPAM Client
2022/01/14 23:18:12 [DEBUG] [MGR] Creating Manager with Provider: f5-ip-provider
2022/01/14 23:18:12 [DEBUG] [STORE] Using IPAM DB file from mount path
2022/01/14 23:18:12 [DEBUG] Added Label: Test
2022/01/14 23:18:12 [DEBUG] Added Label: Production
2022/01/14 23:18:12 [DEBUG] [STORE] [ipaddress status ipam_label reference]
2022/01/14 23:18:12 [DEBUG] [STORE] 10.192.75.113 1 Test Uv38ByGCZU8WP18P
2022/01/14 23:18:12 [DEBUG] [STORE] 10.192.75.114 1 Test lWbHTRADfE17uwQH
2022/01/14 23:18:12 [DEBUG] [STORE] 10.192.75.115 1 Test gYVa2GgdDYbR6R4A
2022/01/14 23:18:12 [DEBUG] [STORE] 10.192.75.116 1 Test ZpTSxCKs0gigByk5
2022/01/14 23:18:12 [DEBUG] [STORE] 10.192.125.30 1 Production 650YpEeEBF2H88Z8
2022/01/14 23:18:12 [DEBUG] [STORE] 10.192.125.31 1 Production la9aJTZ5Ubqi/2zU
2022/01/14 23:18:12 [DEBUG] [STORE] 10.192.125.32 1 Production X7kLrbN8WCG22VUm
2022/01/14 23:18:12 [DEBUG] [STORE] 10.192.125.33 1 Production aAtOfIt2OhsdSdSV
2022/01/14 23:18:12 [DEBUG] [STORE] 10.192.125.34 1 Production YyUlP+xzjdep4ov5
2022/01/14 23:18:12 [DEBUG] [STORE] 10.192.125.35 1 Production DwcCRIYVu9oIMT9q
2022/01/14 23:18:12 [DEBUG] [STORE] 10.192.125.36 1 Production C/UFmHWSHmaKW98s
2022/01/14 23:18:12 [DEBUG] [STORE] 10.192.125.37 1 Production ktJXK80GaNLWxS9Q
2022/01/14 23:18:12 [DEBUG] [STORE] 10.192.125.38 1 Production a/hMcXTLdHY2TMPb
2022/01/14 23:18:12 [DEBUG] [STORE] 10.192.125.39 1 Production Fy7YV5S7NYsMO1Jd
2022/01/14 23:18:12 [DEBUG] [STORE] 10.192.125.40 1 Production /wlCedsZROvXoZ0P
2022/01/14 23:18:12 [DEBUG] [STORE] 10.192.125.41 1 Production JVqlt9RL7ED4TIkr
2022/01/14 23:18:12 [DEBUG] [STORE] 10.192.125.42 1 Production KbAiO+6l9PdDkfRF
2022/01/14 23:18:12 [DEBUG] [STORE] 10.192.125.43 1 Production lAQDdPaSS5jL+HE/
2022/01/14 23:18:12 [DEBUG] [STORE] 10.192.125.44 1 Production jQGRksJCJOLK/Mrj
2022/01/14 23:18:12 [DEBUG] [STORE] 10.192.125.45 1 Production sUMjpryPnn3x2Skz
2022/01/14 23:18:12 [DEBUG] [STORE] 10.192.125.46 1 Production O+pvWzr23gN0NmxH
2022/01/14 23:18:12 [DEBUG] [STORE] 10.192.125.47 1 Production Bn2JvH8B8fVzmBZZ
2022/01/14 23:18:12 [DEBUG] [STORE] 10.192.125.48 1 Production THIVo7U56x5YScYH
2022/01/14 23:18:12 [DEBUG] [STORE] 10.192.125.49 1 Production 9XF6KJomb5dkeYGZ
2022/01/14 23:18:12 [DEBUG] [STORE] 10.192.125.50 1 Production C0s3OXARXoLtb0El
2022/01/14 23:18:12 [DEBUG] [PROV] Provider Initialised
2022/01/14 23:18:12 [INFO] [CORE] Controller started
2022/01/14 23:18:12 [INFO] Starting IPAMClient Informer
```

## Step 3: Configuring CIS CRD to work with F5 IPAM Controller for the following hosts

Provide a ipamLabel in the virtual server CRD. Make your to create latest CIS virtualserver schema which supports ipamLabel

- hostname "myapp.f5demo.com"
```
apiVersion: "cis.f5.com/v1"
kind: VirtualServer
metadata:
  name: f5-demo-myapp
  labels:
    f5cr: "true"
spec:
  host: myapp.f5demo.com
  ipamLabel: Production
  pools:
  - monitor:
      interval: 20
      recv: ""
      send: /
      timeout: 31
      type: http
    path: /
    service: f5-demo
    servicePort: 80
```

- hostname "mysite.f5demo.com"
```
apiVersion: "cis.f5.com/v1"
kind: VirtualServer
metadata:
  name: f5-demo-mysite
  labels:
    f5cr: "true"
spec:
  host: mysite.f5demo.com
  ipamLabel: Test
  pools:
  - monitor:
      interval: 20
      recv: ""
      send: /
      timeout: 31
      type: http
    path: /
    service: f5-demo
    servicePort: 80
```

Deploy the CRD and CRD schema

```
kubectl create -f customresourcedefinitions.yaml
kubectl create -f vs-mysite.yaml
kubectl create -f vs-myapp.yaml
```

crd-example [repo](https://github.com/mdditt2000/k8s-bigip-ctlr/tree/main/user_guides/ipam/crd-example)

Validate the CRDs are created correctly

```
❯ kubectl get crd,vs,externaldns
NAME                                                                        CREATED AT
customresourcedefinition.apiextensions.k8s.io/externaldnses.cis.f5.com      2022-01-15T00:13:43Z
customresourcedefinition.apiextensions.k8s.io/ingresslinks.cis.f5.com       2022-01-15T00:13:43Z
customresourcedefinition.apiextensions.k8s.io/ipams.fic.f5.com              2022-01-06T06:53:53Z
customresourcedefinition.apiextensions.k8s.io/policies.cis.f5.com           2022-01-15T00:13:43Z
customresourcedefinition.apiextensions.k8s.io/tlsprofiles.cis.f5.com        2022-01-15T00:13:43Z
customresourcedefinition.apiextensions.k8s.io/transportservers.cis.f5.com   2022-01-15T00:13:43Z
customresourcedefinition.apiextensions.k8s.io/virtualservers.cis.f5.com     2022-01-15T00:13:43Z

NAME                                      HOST                TLSPROFILENAME   HTTPTRAFFIC   IPADDRESS   IPAMLABEL    IPAMVSADDRESS   STATUS   AGE
virtualserver.cis.f5.com/f5-demo-myapp    myapp.f5demo.com                                               Production   10.192.125.30   Ok       19m
virtualserver.cis.f5.com/f5-demo-mysite   mysite.f5demo.com                                              Test         10.192.75.113   Ok       24m
```

### Logging output from CIS when allocated IP from IPAM and posted to BIG-IP

```
2022/01/15 00:14:46 [DEBUG] [CORE] Allocated IP: 10.192.75.113 for Request:
Hostname: mysite.f5demo.com     Key:    IPAMLabel: Test IPAddr:         Operation: Create
2022/01/15 00:14:46 [DEBUG] Enqueueing on Update: kube-system/k8s-bigip-ctlr-deployment.k8s.ipam
2022/01/15 00:14:46 [DEBUG] Processing Key: &{0xc0002be580 0xc00015c000 Update}
2022/01/15 00:14:46 [DEBUG] Updated: kube-system/k8s-bigip-ctlr-deployment.k8s.ipam with Status. With IP: 10.192.75.113 for Request:
Hostname: mysite.f5demo.com     Key:    IPAMLabel: Test IPAddr:         Operation: Create
2022/01/15 00:20:02 [DEBUG] Enqueueing on Update: kube-system/k8s-bigip-ctlr-deployment.k8s.ipam
2022/01/15 00:20:02 [DEBUG] Processing Key: &{0xc0002bf8c0 0xc0002be580 Update}

2022/01/15 00:20:02 [DEBUG] [CORE] Allocated IP: 10.192.125.30 for Request:
Hostname: myapp.f5demo.com      Key:    IPAMLabel: Production   IPAddr:         Operation: Create
2022/01/15 00:20:02 [DEBUG] Enqueueing on Update: kube-system/k8s-bigip-ctlr-deployment.k8s.ipam
2022/01/15 00:20:02 [DEBUG] Processing Key: &{0xc0004de580 0xc0002bf8c0 Update}
2022/01/15 00:20:02 [DEBUG] Updated: kube-system/k8s-bigip-ctlr-deployment.k8s.ipam with Status. With IP: 10.192.125.30 for Request:
Hostname: myapp.f5demo.com      Key:    IPAMLabel: Production   IPAddr:         Operation: Create
```

### View the F5 IPAM Controller configuration

F5 IPAM Controller creates the following CRD to create the configuration between CIS and IPAM 

```
❯ kubectl get ipams -n kube-system  -oyaml
apiVersion: v1
items:
- apiVersion: fic.f5.com/v1
  kind: IPAM
  metadata:
    creationTimestamp: "2022-01-14T23:44:10Z"
    generation: 3
    managedFields:
    - apiVersion: fic.f5.com/v1
      fieldsType: FieldsV1
      fieldsV1:
        f:status:
          .: {}
          f:IPStatus: {}
      manager: f5-ipam-controller
      operation: Update
      time: "2022-01-15T00:21:32Z"
    - apiVersion: fic.f5.com/v1
      fieldsType: FieldsV1
      fieldsV1:
        f:spec:
          .: {}
          f:hostSpecs: {}
      manager: k8s-bigip-ctlr.real
      operation: Update
      time: "2022-01-15T00:21:32Z"
    name: k8s-bigip-ctlr-deployment.k8s.ipam
    namespace: kube-system
    resourceVersion: "110617667"
    selfLink: /apis/fic.f5.com/v1/namespaces/kube-system/ipams/k8s-bigip-ctlr-deployment.k8s.ipam
    uid: e83eaad4-0faa-4ee7-85d3-1e2889b0a11a
  spec:
    hostSpecs:
    - host: mysite.f5demo.com
      ipamLabel: Test
    - host: myapp.f5demo.com
      ipamLabel: Production
  status:
    IPStatus:
    - host: mysite.f5demo.com
      ip: 10.192.75.113
      ipamLabel: Test
    - host: myapp.f5demo.com
      ip: 10.192.125.30
      ipamLabel: Production
kind: List
metadata:
  resourceVersion: ""
  selfLink: ""
```
