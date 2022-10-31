# Easy F5/NGINX Integrating using OpenShift 4.11

The purpose of this document is to demonstrate how easy it is to integration **F5 BIG-IP and NGINX technologies using OpenShift 4.11**. This guide simplifies the solution by providing examples and step by step guidance using **Operators and OVN-Kubernetes**

F5 BIG-IP and NGINX provides a solutions called **IngressLink** that use both **F5 BIG-IP Container Ingress Services (CIS)** and **NGINX Ingress Controller** deployed in OpenShift 4.11. It’s an elegant control plane solution that offers a unified method of working with both technologies from a single interface—offering the best of **F5 BIG-IP and NGINX** and **fostering better collaboration across NetOps and DevOps teams**. The diagram below demonstrates the architecture

![architecture](https://github.com/mdditt2000/openshift-4-11/blob/main/ingresslink-on-openshift/diagram/2022-10-24_13-38-38.png)

* Demo on YouTube [video]()
* KubeCon [presentation](https://github.com/mdditt2000/openshift-4-11/blob/main/ingresslink-on-openshift/presentation/KubeCon%202022%20Overview.pptx)

On this page you’ll find:

* Links to the GitHub repositories for all the requisite software
* Documentation for the solution(s)

## Configure F5 BIG-IP HA
This document demonstrates **High Availability (HA) F5 BIG-IP's working with OVN-Kubernetes**. Using OVN-Kubernetes with F5 BIG-IP and static routes removes the complexity of creating VXLAN tunnels or using Calico

### Step 1: Deploy OpenShift using OVNKubernetes

Deploy OpenShift Cluster with **networktype** as **OVNKubernetes**. Change the default to **OVNKubernetes** in the install-config.yaml before creating the cluster

### Step 2: Verify gateway mode set to shared

```
# oc get nodes
NAME                        STATUS   ROLES    AGE   VERSION
ocp-pm-trw88-master-0       Ready    master   68m   v1.24.0+3882f8f
ocp-pm-trw88-master-1       Ready    master   67m   v1.24.0+3882f8f
ocp-pm-trw88-master-2       Ready    master   67m   v1.24.0+3882f8f
ocp-pm-trw88-worker-d6zsg   Ready    worker   50m   v1.24.0+3882f8f
ocp-pm-trw88-worker-k7lsd   Ready    worker   55m   v1.24.0+3882f8f
ocp-pm-trw88-worker-vdtmb   Ready    worker   55m   v1.24.0+3882f8f
```

```
# oc logs -f ovnkube-node-2bcx7 ovnkube-node -n openshift-ovn-kubernetes|grep "gateway_mode_flags"
+ gateway_mode_flags='--gateway-mode shared --gateway-interface br-ex'
```

### Step 3: Configure BIG-IP Routes

Configure static routes in BIG-IP with node subnets assigned for the three worker nodes in the OpenShift cluster. Get the node subnet assigned and host address

```
# oc describe node ocp-pm-trw88-worker-d6zsg |grep "node-subnets\|node-primary-ifaddr"
k8s.ovn.org/host-addresses: ["10.192.125.172"]
k8s.ovn.org/node-subnets: {"default":"10.129.2.0/23"}

# oc describe node ocp-pm-trw88-worker-k7lsd |grep "node-subnets\|node-primary-ifaddr"
k8s.ovn.org/node-primary-ifaddr: {"ipv4":"10.192.125.174/24"}
k8s.ovn.org/node-subnets: {"default":"10.131.0.0/23"}

# oc describe node ocp-pm-trw88-worker-vdtmb |grep "node-subnets\|node-primary-ifaddr"
k8s.ovn.org/host-addresses: ["10.192.125.177"]
k8s.ovn.org/node-subnets: {"default":"10.128.2.0/23"}
```

Add static routes to BIG-IP via tmsh for all node subnets using 

```
tmsh create /net route <node_subnet> gw <node_ip>
```
```
tmsh create /net route 10.129.2.0/23 gw 10.192.125.172
tmsh create /net route 10.131.0.0/23 gw 10.192.125.174
tmsh create /net route 10.128.2.0/23 gw 10.192.125.177
```
View static routes created on BIG-IP

![routes](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/ovn-kubernetes-ha/diagram/2022-10-12_13-30-34.png)

**Note:** Manually sync the BIG-IP so the routes are deployed on the standby

### Step 4: Deploy CIS for each BIG-IP

**IngressLink** utilizes a CRD, which creates a port 80 and 443 Virtual Servers on the BIG-IP. The CRD also adds health monitoring which monitors the liveliness of NGINX Ingress Controller. All you need todo is create the VirtualServer CRD. Make sure all the CRD schema's are installed for CIS and NGINX. Also add the proxy protocol iRule if required 

Add the following parameters to the CIS deployment

* Namespace should be **nginx-ingress**
* Enable CRD mode on CIS

### BIG-IP 01

```
args: [
  # See the k8s-bigip-ctlr documentation for information about
  # all config options
  # https://clouddocs.f5.com/containers/latest/
  "--bigip-username=$(BIGIP_USERNAME)",
  "--bigip-password=$(BIGIP_PASSWORD)",
  "--bigip-url=10.192.125.60",
  "--bigip-partition=OpenShift",
  "--namespace=nginx-ingress",
  "--pool-member-type=cluster",
  "--insecure=true",
  "--custom-resource-mode=true",
  "--as3-validation=true",
  "--log-as3-response=true",
]
```

### BIG-IP 02

```
args: [
  # See the k8s-bigip-ctlr documentation for information about
  # all config options
  # https://clouddocs.f5.com/containers/latest/
  "--bigip-username=$(BIGIP_USERNAME)",
  "--bigip-password=$(BIGIP_PASSWORD)",
  "--bigip-url=10.192.125.61",
  "--bigip-partition=OpenShift",
  "--namespace=nginx-ingress",
  "--pool-member-type=cluster",
  "--insecure=true",
  "--custom-resource-mode=true",
  "--as3-validation=true",
  "--log-as3-response=true",
]
```

Deploy CIS in OpenShift

```
oc create secret generic bigip-login -n kube-system --from-literal=username=admin --from-literal=password=<secret>
oc create -f bigip-ctlr-clusterrole.yaml
oc create -f f5-bigip-ctlr-01-deployment.yaml
oc create -f f5-bigip-ctlr-02-deployment.yaml
oc create -f CustomResourceDefinition.yaml
```

CIS [repo](https://github.com/mdditt2000/k8s-bigip-ctlr/tree/main/user_guides/ovn-kubernetes-ha/next-gen-route/cis)

Validate both CIS instances are running 

```
# oc get pod -n kube-system
NAME                                            READY   STATUS    RESTARTS   AGE
k8s-bigip-ctlr-01-deployment-7cc8b7cf94-2csz7   1/1     Running   0          16s
k8s-bigip-ctlr-02-deployment-5c8d8c4676-hjwpr   1/1     Running   0          16s
```

## Deploy NGINX Ingress Operator on OpenShift

Recommend following [NGINX blog](https://github.com/nginxinc/nginx-ingress-helm-operator)

Using the following version:

* NGINX Ingress Controller 2.4.x
* NGINX Ingress Operator 1.2.0

### Step 1: Create the scc resource

 Create the scc resource on the cluster by applying the scc.yaml

```
# oc apply -f scc.yaml
securitycontextconstraints.security.openshift.io/nginx-ingress-admin created
```

SCC [repo](https://github.com/mdditt2000/openshift-4-11/blob/main/ingresslink-on-openshift/nginx-config/scc.yaml)

### Step 2: Create the NginxIngress Instance

Change the name

```
metadata:
  name: nginxingress
```

Also change reportIngressStatus for IngressLink to true

```
reportIngressStatus:
      annotations: {}
      enable: true
      enableLeaderElection: true
      ingressLink: true
```

![operator](https://github.com/mdditt2000/openshift-4-11/blob/main/ingresslink-on-openshift/diagram/2022-10-31_11-04-07.png)

### Step 3: Validate NGINX Ingress Operator on OpenShift

Make sure the **service/nginxingress-nginx-ingress** and **pod/nginxingress-nginx-ingress** are running as shown below

```
[root@ocp-installer crd-resource]#  oc -n nginx-ingress get all
NAME                                                             READY   STATUS    RESTARTS   AGE
pod/nginx-ingress-operator-controller-manager-79466fc4f8-b28g4   2/2     Running   0          4d2h
pod/nginxingress-nginx-ingress-8597774775-77gn5                  1/1     Running   0          4d2h

NAME                                                                TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
service/nginx-ingress-operator-controller-manager-metrics-service   ClusterIP      172.30.215.139   <none>        8443/TCP                     4d2h
service/nginxingress-nginx-ingress                                  LoadBalancer   172.30.66.55     <pending>     80:30971/TCP,443:31260/TCP   4d2h

NAME                                                        READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/nginx-ingress-operator-controller-manager   1/1     1            1           4d2h
deployment.apps/nginxingress-nginx-ingress                  1/1     1            1           4d2h

NAME                                                                   DESIRED   CURRENT   READY   AGE
replicaset.apps/nginx-ingress-operator-controller-manager-79466fc4f8   1         1         1       4d2h
replicaset.apps/nginxingress-nginx-ingress-8597774775                  1         1         1       4d2h
[root@ocp-installer crd-resource]#
s
```

### Step 4: Add label to the Service

Add label **app=nginxingress-nginx-ingress** to the **nginxingress-nginx-ingress**. This is required for CIS service discovery

![service](https://github.com/mdditt2000/openshift-4-11/blob/main/ingresslink-on-openshift/diagram/2022-10-31_11-14-49.png)

### Step 5: Add egress from OpenShift cluster to BIG-IP

Add annotation on **nginx-ingress** namespace for egress from OpenShift cluster to BIG-IP using k8s.ovn.org/routing-external-gws annotation. Use the BIG-IP floating self-IP address for the **routing-external-gws: 10.192.125.62**

![name-space](https://github.com/mdditt2000/openshift-4-11/blob/main/ingresslink-on-openshift/diagram/2022-10-31_11-19-38.png)

## Deploy the Cafe Application

### Step 1: Configure Load Balancing for the Cafe Application

Create the coffee and the tea deployments and services:

    kubectl create -f cafe.yaml

Create a secret with an SSL certificate and a key:

    kubectl create -f cafe-secret.yaml

Create an Ingress resource:

    kubectl create -f cafe-ingress.yaml

demo application [repo](https://github.com/mdditt2000/k8s-bigip-ctlr/tree/main/user_guides/ingresslink-externaldns/ingress-example)

## Create the CRD Resource

```
# oc create -f vs-ingresslink.yaml
ingresslink.cis.f5.com/vs-ingresslink created
```

CRD [repo](https://github.com/mdditt2000/openshift-4-11/blob/main/ingresslink-on-openshift/cis/crd-resource/vs-ingresslink.yaml)

## Connect to the VirtualServer

* BIG-IP is connecting directly to the application pod
* Real-IP is the BIG-IP floating IP

![application](https://github.com/mdditt2000/openshift-4-11/blob/main/ingresslink-on-openshift/diagram/2022-10-31_11-24-40.png)