# OpenShift Routes verse Custom Resource Definitions

This document compares the differences between OpenShift Routes and Custom Resource Definitions. Custom Resource allows you to extend Kubernetes capabilities by adding any kind of API object useful for your application. Custom Resource Definition is what you use to define a Custom Resource. This is a powerful way to extend F5 CIS Kubernetes capabilities beyond the default installation and similar to OpenShift Routes.

In this example we are using a cafe application with three endpoints; **tea,coffee and mocha** as shown in the diagram below

![architecture](https://github.com/mdditt2000/openshift-4-9/blob/main/route-vs-crd/diagram/2022-01-26_14-39-29.png)

Demo on YouTube [video]()

## Using OpenShift Route

### Step 1: Deploy CIS

Currently in CIS 2.7 only one Public IP **Virtual Server** for BIG-IP can be configured for all Routes. Routes uses HOST Header Load balancing to determine the backend
application. In this example the backend is **/tea,/coffee and /mocha**

Add the following parameters to the CIS deployment

* --route-vserver-addr=10.192.125.65 - Public IP for BIG-IP for all Routes
* --manage-routes=true - Configure CIS to watch for Routes
* --bigip-partition=OpenShift - CIS uses BIG-IP tenant OpenShift to manage Routes
* --openshift-sdn-name=/Common/openshift_vxlan - CNI policy on BIG-IP to connect to the PODs in OpenShift
* --override-as3-declaration=default/cafe-override - AS3 Override allows you to add Objects not exposed by an annotations such as custom policies, iRules, profiles etc

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
  "--openshift-sdn-name=/Common/openshift_vxlan",
  "--insecure=true",
  "--manage-routes=true",
  "--route-vserver-addr=10.192.125.65",
  "--override-as3-declaration=default/cafe-override",
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

CIS [repo](https://github.com/mdditt2000/openshift-4-9/tree/main/route-vs-crd/route/cis)

**Note** Do not forget the CNI [repo](https://github.com/mdditt2000/openshift-4-9/tree/main/route-vs-crd/route/cni)

### Step 2: Creating OpenShift Routes

User-case for the OpenShift Routes:

- Edge Termination
- Redirect HTTP to HTTPS
- Health monitor of the backend NGINX application using HOST **cafe.example.com** and **PATH /coffee, /tea and /mocha**
- Custom HTTP Policy for X-Forwarded-For (XFF) HTTP header field - **Use AS3 override**
- Backend listening on PORT 8080

**Note** No annotations for custom HTTP Policy under the Virtual Server. You need to use AS3 override configmap to add custom policies etc

Diagram blow displays the example of **route-tea** with the **AS3 override** for the following use-case

![route-as3-override](https://github.com/mdditt2000/openshift-4-9/blob/main/route-vs-crd/diagram/2022-01-27_11-23-54.png)

Create OpenShift Routes

```
oc create -f cafe-override.yaml
oc create -f route-tea.yaml
oc create -f route-coffee.yaml
oc create -f route-mocha.yaml
```
Routes [repo](https://github.com/mdditt2000/openshift-4-9/tree/main/route-vs-crd/route/ocp-route)

Validate OpenShift Routes using the OpenShift Dashboard

![route](https://github.com/mdditt2000/openshift-4-9/blob/main/route-vs-crd/diagram/2022-01-27_14-39-54.png)

Validate OpenShift Routes using the BIG-IP

![big-ip route](https://github.com/mdditt2000/openshift-4-9/blob/main/route-vs-crd/diagram/2022-01-27_11-36-07.png)

Validate OpenShift Routes pool-members using the BIG-IP

![big-ip pools](https://github.com/mdditt2000/openshift-4-9/blob/main/route-vs-crd/diagram/2022-01-27_11-38-40.png)

Validate OpenShift Routes by connecting to the Public IP

![traffic](https://github.com/mdditt2000/openshift-4-9/blob/main/route-vs-crd/diagram/2022-01-27_11-44-57.png)

## Using Custom Resource Definitions

### Step 1: Deploy CIS

The biggest benefit for using CRDs is their no limitations on how many Public IPs **Virtual Server** create on BIG-IP. However to maintain similarity with Routes we using HOST Header Load balancing to determine the backend application. In this example the backend is **/tea,/coffee and /mocha** with the same Public IP address 10.192.125.65

Add the following parameters to THE CIS deployment

* --custom-resource-mode=true - Configure CIS to watch for CRDs
* --bigip-partition=OpenShift - CIS uses BIG-IP tenant OpenShift to manage CRDs
* --openshift-sdn-name=/Common/openshift_vxlan - CNI policy on BIG-IP to connect to the PODs in OpenShift

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
  "--openshift-sdn-name=/Common/openshift_vxlan",
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
oc create -f f5-bigip-ctlr-deployment.yaml
```

CIS [repo](https://github.com/mdditt2000/openshift-4-9/tree/main/route-vs-crd/customresource/cis)

**Note** Do not forget the CNI [repo](https://github.com/mdditt2000/openshift-4-9/tree/main/route-vs-crd/route/cni)

### Step 2: Creating Custom Resource Definitions

Similar User-case for the CRDs:

- Edge Termination
- Redirect HTTP to HTTPS
- Health monitor of the backend NGINX application using HOST **cafe.example.com** and **PATH /coffee, /tea and /mocha**
- Custom HTTP Policy for X-Forwarded-For (XFF) HTTP header field
- Backend listening on PORT 8080

**Note** Unlike OpenShift Routes, CIS can reference dedicated CRDs for specific tasks such as ExternalDNS, Policies, TLS Termination etc. 

Diagram below displays the example of **vs-tea** with the **dge-tls** and **cafe-policy** for the following use-case. The TLS, Policy CRD are referenced in the VirtualServer CRD as shown in the diagram below

![crd-policy-tls](https://github.com/mdditt2000/openshift-4-9/blob/main/route-vs-crd/diagram/2022-01-27_13-37-29.png)

Create OpenShift CRDs

**Note:** CIS requires the CustomResourceDefinition schema

```
oc create -f CustomResourceDefinition.yaml
```

CRD Schema [repo](https://github.com/mdditt2000/openshift-4-9/blob/main/route-vs-crd/customresource/crd/crd-schema/customresourcedefinitions.yml)

Create OpenShift CRDs

```
oc create -f edge-tls.yaml
oc create -f cafe-policy.yaml
oc create -f vs-tea.yaml
oc create -f vs-coffee.yaml
oc create -f vs-mocha.yaml
```

CRD [repo](https://github.com/mdditt2000/openshift-4-9/tree/main/route-vs-crd/customresource/crd)

Validate OpenShift Routes using from OpenShift

**Note** Sadly OpenShift does not have the same Dashboard for CRDs. Therefore you need to use the OpenShift CLI

```
# oc get crd,vs,policy,tlsprofile -n default
NAME                                   HOST               TLSPROFILENAME   HTTPTRAFFIC   IPADDRESS       IPAMLABEL   IPAMVSADDRESS   STATUS   AGE
virtualserver.cis.f5.com/cafe-coffee   cafe.example.com   edge-tls         redirect      10.192.125.65                                        68s
virtualserver.cis.f5.com/cafe-mocha    cafe.example.com   edge-tls         redirect      10.192.125.65                                        68s
virtualserver.cis.f5.com/cafe-tea      cafe.example.com   edge-tls         redirect      10.192.125.65               None            Ok       68s

NAME                            AGE
policy.cis.f5.com/cafe-policy   68s

NAME                             AGE
tlsprofile.cis.f5.com/edge-tls   68s
```

Validate OpenShift Routes using the BIG-IP

![big-ip CRD](https://github.com/mdditt2000/openshift-4-9/blob/main/route-vs-crd/diagram/2022-01-27_14-47-04.png)

Validate OpenShift CRD pool-members using the BIG-IP

![big-ip pools](https://github.com/mdditt2000/openshift-4-9/blob/main/route-vs-crd/diagram/2022-01-27_14-51-23.png)

Validate OpenShift CRD by connecting to the Public IP

![traffic](https://github.com/mdditt2000/openshift-4-9/blob/main/route-vs-crd/diagram/2022-01-27_11-44-57.png)
