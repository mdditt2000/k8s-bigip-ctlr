# RedHat OpenShift OVN-Kubernetes using Routes for F5 BIG-IP Standalone

This document demonstrates how to use **OVN-Kubernetes with F5 BIG-IP Routes** to Ingress traffic without using an Overlay. Using OVN-Kubernetes with F5 BIG-IP Routes removes the complexity of creating VXLAN tunnels or using Calico. This document demonstrates **Standalone BIG-IP working with OVN-Kubernetes**. Diagram below demonstrates a OpenShift 4.11 Cluster with three masters and three worker nodes. The three applications; **tea,coffee and mocha** are deployed in the **cafe** namespace. As you can see from the diagram below the **cafe** namespace requires an annotation for the BIG-IP self-ip. 

![architecture](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/ovn-kubernetes-standalone/diagram/2022-10-12_12-48-49.png)

Demo on YouTube [video]()

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

![routes](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/ovn-kubernetes-standalone/diagram/2022-10-12_13-30-34.png)

### Step 4: Configure BIG-IP Routes

Configure egress from OpenShift cluster to BIG-IP using k8s.ovn.org/routing-external-gws annotation on namespace where the application is deployed as shown in the diagram above

```
apiVersion: v1
kind: Namespace
metadata:
  annotations:
    k8s.ovn.org/routing-external-gws: 10.192.125.60 ##BIG-IP interface address rotatable to the OpenShift nodes
  labels:
    kubernetes.io/metadata.name: default
  name: cafe
```
routing-external-gws [repo](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/ovn-kubernetes-standalone/demo-app/cafe/name-cafe.yaml)

**Setup complete!** Deploy CIS and create OpenShift Routes

### Step 5: Deploy CIS

F5 Controller Ingress Services (CIS) called **Next Generation Routes Controller**. Next Generation Routes Controller extended F5 CIS to use multiple Virtual IP addresses. Before F5 CIS could only manage one Virtual IP address per CIS instance.

Add the following parameters to the CIS deployment

* Routegroup specific config for each namespace is provided as part of extendedSpec through ConfigMap.
* ConfigMap info is passed to CIS with argument --route-spec-configmap="namespace/configmap-name"
* Controller mode should be set to openshift to enable multiple VIP support(--controller-mode="openshift")

```
args: [
  # See the k8s-bigip-ctlr documentation for information about
  # all config options
  # https://clouddocs.f5.com/containers/latest/
  "--bigip-username=$(BIGIP_USERNAME)",
  "--bigip-password=$(BIGIP_PASSWORD)",
  "--bigip-url=10.192.125.60",
  "--bigip-partition=OpenShift",
  "--namespace=default",
  "--pool-member-type=cluster",
  "--insecure=true",
  "--manage-routes=true",
  "--route-spec-configmap="kube-system/global-cm"
  "--controller-mode="openshift"
  "--as3-validation=true",
  "--log-as3-response=true",
]
```

Deploy CIS in OpenShift

```
oc create secret generic bigip-login -n kube-system --from-literal=username=admin --from-literal=password=<secret>
oc create -f bigip-ctlr-clusterrole.yaml
oc create -f f5-bigip-ctlr-deployment.yaml
```

CIS [repo](https://github.com/mdditt2000/k8s-bigip-ctlr/tree/main/user_guides/ovn-kubernetes-standalone/next-gen-route/cis)

### Step 6: Deploy Global ConfigMap

Using Global ConfigMap

* Global ConfigMap provides control to the admin to create and maintain the resource configuration centrally.
* namespace: cafe, vserverAddr: 10.192.125.65

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: global-cm
  namespace: kube-system
  labels:
    f5nr: "true"
data:
  extendedSpec: |
    extendedRouteSpec:
    - namespace: cafe
      vserverAddr: 10.192.125.65
      vserverName: cafe
      allowOverride: true
```

Deploy global ConfigMap

```
oc create -f global-cm.yaml
```
ConfigMap [repo](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/ovn-kubernetes-standalone/next-gen-route/route/global-cm.yaml)

### Step 6 Creating OpenShift Routes for cafe.example.com

User-case for the OpenShift Routes:

- Edge Termination
- Backend listening on PORT 8080

Create OpenShift Routes

```
oc create -f route-tea-edge.yaml
oc create -f route-coffee-edge.yaml
oc create -f route-mocha-edge.yaml
```

Routes [repo](https://github.com/mdditt2000/k8s-bigip-ctlr/tree/main/user_guides/ovn-kubernetes-standalone/next-gen-route/route/cafe/secure)

Validate OpenShift Routes using the BIG-IP

![big-ip route](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/ovn-kubernetes-standalone/diagram/2022-06-07_15-35-21.png)

Validate OpenShift Virtual IP using the BIG-IP

![big-ip pools](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/ovn-kubernetes-standalone/diagram/2022-06-07_15-37-33.png)

Validate OpenShift Routes policies on the BIG-IP

![traffic](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/ovn-kubernetes-standalone/diagram/2022-06-07_15-38-08.png)

Validate OpenShift Routes policies by connecting to the Public IP

![traffic](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/ovn-kubernetes-standalone/diagram/2022-10-12_13-46-30.png)