# Advancing OpenShift Routes using ExternalDNS and WAF

This document demonstrates how F5 Controller Ingress Services (CIS) can advance OpenShift Routes. F5 CIS can expand the OpenShift Route API to use multiple Public IP addresses per Host. Without F5 CIS, OpenShift Route API can only manage one Public IP address. 

In this example we are using multiple hosts **cafeone** and **cafetwo** with three endpoints; **tea,coffee and mocha** as shown in the diagram below. Wide-IPs for Hosts **cafeone.example.com** and **cafetwo.example.com** are created on F5 GTM using ExternalDNS CRDs. All application will be protected using F5 WAF. All the Routes, ExternalDNS and WAF is managed from the OpenShift API.

![architecture](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/routes-externaldns-waf/diagram/2022-12-06_15-50-57.png)

Demo on YouTube [video](https://youtu.be/ZJDw8i_ZegM)

## Using CIS Next Generation OpenShift Route

### Step 1: Deploy CIS

Currently OpenShift Route API using HAproxy which can only support one **Public IP**, **Virtual Server** per BIG-IP. Routes uses HOST Header Load balancing to determine the backend application. CIS can resolve this issue by using a Global ConfigMap to specify global objects like IPs, Policies etc. This is similar to how Gateway API will work. The Global ConfigMap can be applied in different namespace than the Routes

Add the following parameters to the CIS deployment

* Global ConfigMap info is passed to CIS with argument --route-spec-configmap="namespace/configmap-name"
* Controller mode should be set to openshift to enable multiple VIP support(--controller-mode="openshift")
* Using AS3 API to post ExternalDNS configuration --cccl-gtm-agent=false
* Using OVN Kubernetes with static routes. No CNI configured

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
    "--namespace=cafeone",
    "--namespace=cafetwo",
    "--pool-member-type=cluster",
    "--log-level=DEBUG",
    "--insecure=true",
    "--route-spec-configmap=default/global-cm",
    "--controller-mode=openshift",
    "--as3-validation=true",
    "--log-as3-response=true",
    "--cccl-gtm-agent=false",
```

Deploy CIS in OpenShift

```
oc create secret generic bigip-login -n kube-system --from-literal=username=admin --from-literal=password=<secret>
oc create -f bigip-ctlr-clusterrole.yaml
oc create -f f5-bigip-ctlr-deployment.yaml
```

CIS [repo](https://github.com/mdditt2000/k8s-bigip-ctlr/tree/main/user_guides/routes-externaldns-waf/cis)

### Step 2: Deploy Global ConfigMap

Using Global ConfigMap

* Global ConfigMap provides control to the admin to create and maintain the resource configuration centrally like Public IPs, Policies, Certificates etc
* RBAC can be used to restrict modification of global ConfigMap by users with tenant level access

* namespace: cafeone, vserverAddr: **10.192.125.65**
* namespace: cafetwo, vserverAddr: **10.192.125.66**

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: global-cm
  namespace: default
  labels:
    f5nr: "true"
data:
  extendedSpec: |
    extendedRouteSpec:
    - namespace: cafeone
      vserverAddr: 10.192.125.65
      vserverName: cafeone
      allowOverride: true
      policyCR: default/policy-cafe
    - namespace: cafetwo
      vserverAddr: 10.192.125.66
      vserverName: cafetwo
      allowOverride: true
      policyCR: default/policy-cafe
```

Deploy global ConfigMap

```
oc create -f global-cm.yaml
```

### Step 3: Creating OpenShift Routes for cafe.example.com

User-case for the OpenShift Routes:

- Edge Termination
- Backend listening on PORT 8080
- Wide-IP configured to GTM
- WAF, logging policies applied to Virtual IP via PolicyCR

Create OpenShift Routes for **cafeone.example.com**

```
oc create -f route-tea-edge.yaml
oc create -f route-coffee-edge.yaml
oc create -f route-mocha-edge.yaml
```
Routes [repo](https://github.com/mdditt2000/k8s-bigip-ctlr/tree/main/user_guides/routes-externaldns-waf/ocp-route/cafeone/route)

Create OpenShift Routes for **cafetwo.example.com**

```
oc create -f route-tea-edge.yaml
oc create -f route-coffee-edge.yaml
oc create -f route-mocha-edge.yaml
```
Routes [repo](https://github.com/mdditt2000/k8s-bigip-ctlr/tree/main/user_guides/routes-externaldns-waf/ocp-route/cafetwo)

Validate OpenShift Routes for cafeone

![openshift](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/routes-externaldns-waf/diagram/2022-12-06_16-49-29.png)

Validate OpenShift Virtual IPs for **cafeone** and **cafetwo** using the BIG-IP

![big-ip](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/routes-externaldns-waf/diagram/2022-12-06_16-50-18.png)

### Step 4: Creating Wide-IPs on GTM

* AS3-40 is a requirement
* Create DataCenter and Server in the Common partition
* VirtualServer Discovery is required

In this example I created the GTM global objects using AS3

AS3 [repo](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/routes-externaldns-waf/bigip-gslb-common/bigip-gslb-common.json)

Create Wide-IP for **cafeone.example.com**

```
# oc create -f edns-cafeone.yaml
externaldns.cis.f5.com/edns-cafe created
```

ExternalDNS [repo](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/routes-externaldns-waf/ocp-route/cafeone/crd/edns-cafeone.yaml)

Create Wide-IP for **cafetwo.example.com**

```
# oc create -f edns-cafetwo.yaml
externaldns.cis.f5.com/edns-cafe created
```

ExternalDNS [repo](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/routes-externaldns-waf/ocp-route/cafetwo/crd/edns-cafetwo.yaml)

Validate Wide-IPs on the BIG-IP

![WIDE-IP](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/routes-externaldns-waf/diagram/2022-12-06_17-03-01.png)

### Step 4: Enable WAF Protection for the OpenShift Cluster

Global ConfigMap allows for Policies to be associated to the routes. In my example both hosts **cafeone** and **cafetwo**  are using the same WAF policies. CIS uses AS3 simply references and existing policy on BIG-IP. Logging of all request is also enabled. The PolicyCRD provides flexibility of adding multiple mandatory configures requested on BIG-IP that not exposed by OpenShift Routes API or annotations 

**Global ConfigMap**

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: global-cm
  namespace: default
  labels:
    f5nr: "true"
data:
  extendedSpec: |
    extendedRouteSpec:
    - namespace: cafeone
      vserverAddr: 10.192.125.65
      vserverName: cafeone
      allowOverride: true
      policyCR: default/policy-cafe ----- reference to PolicyCRD
    - namespace: cafetwo
      vserverAddr: 10.192.125.66
      vserverName: cafetwo
      allowOverride: true
      policyCR: default/policy-cafe ----- reference to PolicyCRD
```

**PolicyCRD**

```
apiVersion: cis.f5.com/v1
kind: Policy
metadata: 
  labels: 
    f5cr: "true"
  name: policy-cafe
  namespace: default
spec:
  l7Policies:
    waf: /Common/WAF_Policy
  profiles: 
    logProfiles: 
      - /Common/Log all requests
```

**Note** CIS CRD schema is required

Create PolicyCRD

```
# oc create -f policy-cafe.yaml
```

PolicyCRD [repo](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/routes-externaldns-waf/ocp-route/cafeone/crd/policy-cafe.yaml)

Validate WAF Policy on the BIG-IP

![WAF](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/routes-externaldns-waf/diagram/2022-12-06_17-39-38.png)

### Step 5: Prevent Malicious traffic from OpenShift Cluster

Enable a XSS attacks and make sure WAF blocks the Malicious traffic

![WAF](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/routes-externaldns-waf/diagram/2022-12-06_17-45-12.png)

Validate Request Logging on BIG-IP 

![Block](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/routes-externaldns-waf/diagram/2022-12-06_17-50-16.png)