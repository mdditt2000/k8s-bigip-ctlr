## OpenShift Ingress in a Multi-Cluster World with NGINX + BIG-IP

This document demonstrates how **NGINX Ingress Controller** can help scale, secure, and provide visibility into OpenShift in a **multi-cluster world**. The YouTube demo demonstrates OpenShift Multi-Cluster using F5 BIG-IP and Container Ingress Services (CIS) and NGINX IC. This document focuses on **standalone deployment** using **ClusterIP**. CIS is only deployed in OpenShift 4.11 cluster as shown in the diagram. However CIS can also be deployed in multi-cluster using **active/standby** or **active/active**. This will be covered in another demo

![architecture](hhttps://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/multi-cluster-nginx/diagram/2023-08-16_11-51-42.png)


YouTube [Demo]()

## Why OpenShift Multi-Cluster

My environment has two cluster as shown in the diagram above:

* OpenShift 4.11
* OpenShift 4.13
* Similar Pods deployed in both cluster load-balanced by NGINX Ingress Controller
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

I would like to distribute traffic between the two cluster. While evaluating OpenShift 4.13 before upgrading/rebuilding OpenShift 4.11. Steps used to configure OpenShift multi-cluster using CIS **standalone deployment**.

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
CIS deployment [repo](https://github.com/mdditt2000/k8s-bigip-ctlr/tree/main/user_guides/multi-cluster-nginx/openshift-4-11/cis)

**OpenShift-4-13 Cluster**

Create RBAC for CIS. This RBAC is created in OpenShift-4-13

```
# oc create -f external-cluster-rabc.yaml
```

RBAC [repo](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/multi-cluster-nginx/openshift-4-13/cis/external-cluster-rabc.yaml)

#### Step 3 Deploy NGINX Ingress Controller in both Clusters

Deploy the NGINX Ingress Controller in **OpenShift-4-11** for ClusterIP

Getting Started [repo](https://github.com/nginxinc/nginx-ingress-helm-operator#getting-started)

View Service in **OpenShift-4-11** for NGINX-Ingress

```
# oc get service -n nginx-ingress
NAME                                                        TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
nginx-ingress-operator-controller-manager-metrics-service   ClusterIP   172.30.209.54   <none>        8443/TCP         4d23h
nginxingress-sample-nginx-ingress-controller                ClusterIP   172.30.69.213   <none>        80/TCP,443/TCP   4d22h
```

Deploy the NGINX Ingress Controller in **OpenShift-4-13** for ClusterIP

Getting Started [repo](https://github.com/nginxinc/nginx-ingress-helm-operator#getting-started)

View Service in **OpenShift-4-13** for NGINX-Ingress

```
# oc get service -n nginx-ingress
NAME                                                        TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
nginx-ingress-operator-controller-manager-metrics-service   ClusterIP   172.30.209.54   <none>        8443/TCP         4d23h
nginxingress-sample-nginx-ingress-controller                ClusterIP   172.30.69.213   <none>        80/TCP,443/TCP   4d22h
```

#### Step 3 Deploy OpenShift Route for NGINX-Ingress traffic

**Note:** Global ConfigMap creates the gateway **BIG-IP** configuration from HA, Public IP Addresses required for Hostname. 

Create global ConfigMap in **OpenShift-4-11**

```
# oc get configmap global-cm -n default
NAME        DATA   AGE
global-cm   1      2d
```

Global ConfigMap [repo](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/multi-cluster-nginx/openshift-4-11/extendedConfigMap/global-spec-config.yaml)

Create Route in **OpenShift-4-11**

```
[root@ocp-installer openshift-4-11]# oc get route -n nginx-ingress
NAME                      HOST/PORT                     PATH   SERVICES                                       PORT   TERMINATION            WILDCARD
nginx-ingress-route-443   cafe.example.com ... 1 more          nginxingress-sample-nginx-ingress-controller   443    passthrough/Redirect   None
```

Cafe Routes [repo](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/multi-cluster-nginx/openshift-4-11/ocp-route/nginx-ingress/nginx-ingress-route-443.yaml)

BIG-IP Pools members show the Pods from both **OpenShift-4-11** and **OpenShift-4-13**

![pods](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/multi-cluster-nginx/diagram/2023-08-17_09-23-05.png)
