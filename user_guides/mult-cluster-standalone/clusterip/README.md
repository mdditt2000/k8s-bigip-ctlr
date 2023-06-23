## ## OpenShift Multi-Cluster Standalone using ClusterIP

This document demonstrates OpenShift Multi-Cluster using F5 BIG-IP. This document focuses on **standalone deployment** using **ClusterIP**. Container Ingress Services (CIS) is only deployed in OpenShift 4.11 cluster as shown in the diagram

![architecture](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/mult-cluster-standalone/clusterip/diagram/2023-06-14_15-14-41.png)

Demo on YouTube [video]()

## Why OpenShift Multi-Cluster

My environment has two cluster as shown in the diagram above:

* OpenShift 4.11
* OpenShift 4.13
* Similar Pods deployed in both cluster
* Unique Pod Networks using OVNKubernetes

**OpenShift-4-11 Cluster**
```
Spec:
  Cluster Network:
    Cidr:         10.128.0.0/14
    Host Prefix:  23
  External IP:
    Policy:
  Network Type:  OVNKubernetes
  Service Network:
    172.30.0.0/16
```

**OpenShift-4-13 Cluster**
```
Spec:
  Cluster Network:
    Cidr:         10.148.0.0/14
    Host Prefix:  23
  External IP:
    Policy:
  Network Type:  OVNKubernetes
  Service Network:
    172.30.0.0/16
```

node-subnets\|node-primary-ifaddr [list](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/mult-cluster-standalone/clusterip/openshift-4-11/node.md)

I would like to distribute traffic between the two cluster. While evaluating OpenShift 4.13 before upgrading/rebuilding OpenShift 4.11. Steps used to configure OpenShift multi-cluster using CIS **standalone deployment**. CIS configured in HA deployment will come next. 

#### Step 1 Fetch KubeConfigs from OpenShift-4-13 Cluster

Copy the KubeConfig from **OpenShift-4-13** to **OpenShift-4-11** so that CIS can monitor the services in remote clusters. CIS uses KubeClient

**OpenShift-4-11 Cluster**

View the copy of KubeConfig for both clusters

```
[root@ocp-installer auth]# ls
kubeadmin-password  kubeconfig  kubeconfig-openshift-4-13
[root@ocp-installer auth]#
```

Create secrets using the following command 

```
oc create secret generic openshift-4-13 --from-file=kubeconfig=/openshift/ipi/auth/kubeconfig-openshift-4-13
secret/openshift-4-13 created
```

```
# oc get secret
NAME                       TYPE                                  DATA   AGE
openshift-4-11             Opaque                                1      43h
openshift-4-13             Opaque                                1      35h
[root@ocp-installer route]#
```

**Note:** Since CIS is only deployed in OpenShift-4-11, no need to share KubeConfig  with OpenShift-4-13

#### Step 2 Deploy CIS and RBAC

Deploy CIS and RBAC for OpenShift 4-11 and OpenShift 4-13

**OpenShift-4-11 Cluster**

```
# oc create -f bigip-ctlr-clusterrole.yaml
# oc create -f f5-bigip-ctlr-deployment.yaml
deployment.apps/k8s-bigip-ctlr-deployment created
```
CIS deployment [repo](https://github.com/mdditt2000/k8s-bigip-ctlr/tree/main/user_guides/mult-cluster-standalone/clusterip/openshift-4-11/cis)

**OpenShift-4-13 Cluster**

Create RBAC for CIS. This RBAC is created in OpenShift-4-13

```
# oc create -f external-cluster-rabc.yaml
```

RBAC [repo](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/mult-cluster-standalone/clusterip/openshift-4-13/cis/external-cluster-rabc.yaml)

#### Step 3 Deploy Cafe application in both Clusters

Deploy the Cafe Pods, Services using NodePort in **OpenShift-4-11**

Cafe App [repo](https://github.com/mdditt2000/k8s-bigip-ctlr/tree/main/user_guides/mult-cluster-standalone/clusterip/openshift-4-11/demo-app/cafeone)

View Service in **OpenShift-4-11**

```
[root@ocp-installer cafeone]# oc get service -n cafeone
NAME         TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
coffee-svc   ClusterIP   172.30.223.248   <none>        8080/TCP   26h
mocha-svc    ClusterIP   172.30.110.9     <none>        8080/TCP   26h
tea-svc      ClusterIP   172.30.50.211    <none>        8080/TCP   26h
```

**Note:** Ports needs to match the BIG-IP Pools members

Deploy the Cafe Pods, Services using NodePort in **OpenShift-4-13**

Cafe App [repo](https://github.com/mdditt2000/k8s-bigip-ctlr/tree/main/user_guides/mult-cluster-standalone/clusterip/openshift-4-13/demo-app/cafeone)

View Service in **OpenShift-4-13**

```
[root@ocp-installer cafeone]#  oc get service -n cafeone
[root@ocp-installer openshift-4-13]# oc get service -n cafeone
NAME         TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
coffee-svc   ClusterIP   172.30.240.56    <none>        8080/TCP   42m
mocha-svc    ClusterIP   172.30.169.113   <none>        8080/TCP   42m
tea-svc      ClusterIP   172.30.8.131     <none>        8080/TCP   42m
```

#### Step 3 Deploy OpenShift Route

Create routes in **OpenShift-4-11**

```
[root@ocp-installer cafeone]# oc get route -n cafeone
NAME               HOST/PORT                        PATH      SERVICES     PORT   TERMINATION   WILDCARD
cafe-coffee-edge   cafeone.example.com ... 1 more   /coffee   coffee-svc   8080                 None
cafe-mocha-edge    cafeone.example.com ... 1 more   /mocha    mocha-svc    8080                 None
cafe-tea-edge      cafeone.example.com ... 1 more   /tea      tea-svc      8080                 None
```

Cafe Routes [repo](https://github.com/mdditt2000/k8s-bigip-ctlr/tree/main/user_guides/mult-cluster-standalone/clusterip/openshift-4-11/ocp-route/cafeone/nonsecure)

BIG-IP Pools members show the Pods from both **OpenShift-4-11** and **OpenShift-4-13**

![pods](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/mult-cluster-standalone/clusterip/diagram/2023-06-22_16-52-50.png)
