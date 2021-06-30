# OpenShift 4.7 and F5 Container Ingress Services (CIS) User-Guide for Standalone BIG-IP 

This user guide is create to document OpenShift 4.7 integration of CIS and standalone BIG-IP. This user guide provides configuration for a standalone BIG-IP using **OpenShift SDN**

![diagram](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/openshift-4-7/standalone/diagram/2021-06-30_13-40-08.png)

Demo on YouTube [video](https://www.youtube.com/watch?v=-HLcHH_vQJE)

## Environment parameters

* OpenShift 4.7 on vSphere with installer-provisioned infrastructure
* CIS 2.5
* AS3: 3.28
* BIG-IP 16.0.1.1

## Prerequisite

Since CIS is using the AS3 declarative API we need the AS3 extension installed on BIG-IP. Follow the link to install AS3
 
* Install AS3 on BIG-IP
https://clouddocs.f5.com/products/extensions/f5-appsvcs-extension/latest/userguide/installation.html

## Create a BIG-IP VXLAN tunnel for OpenShift SDN

### Step 1:

```
(tmos)# create net tunnels vxlan vxlan-mp flooding-type multipoint
(tmos)# create net tunnels tunnel openshift_vxlan key 0 profile vxlan-mp local-address 10.192.125.60
```

![diagram](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/openshift-4-7/standalone/diagram/2021-06-30_14-35-58.png)

### Step 2:

Create a host subnet for the BIP-IP. This will provide the subnet for creating the tunnel self-IP

    oc create -f f5-openshift-hostsubnet.yaml

```
# oc get hostsubnet
NAME                        HOST                        HOST IP          SUBNET          EGRESS CIDRS   EGRESS IPS
f5-server                   f5-server                   10.192.125.60   10.128.4.0/23
ocp-pm-bwmmz-master-0       ocp-pm-bwmmz-master-0       10.192.75.229   10.130.0.0/23
ocp-pm-bwmmz-master-1       ocp-pm-bwmmz-master-1       10.192.75.231   10.129.0.0/23
ocp-pm-bwmmz-master-2       ocp-pm-bwmmz-master-2       10.192.75.230   10.128.0.0/23
ocp-pm-bwmmz-worker-9ch4b   ocp-pm-bwmmz-worker-9ch4b   10.192.75.234   10.129.2.0/23
ocp-pm-bwmmz-worker-lws6s   ocp-pm-bwmmz-worker-lws6s   10.192.75.235   10.131.0.0/23
ocp-pm-bwmmz-worker-qdhgx   ocp-pm-bwmmz-worker-qdhgx   10.192.75.233   10.128.2.0/23

```
f5-openshift-hostsubnet.yaml [repo](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/openshift-4-7/standalone/cis/f5-openshift-hostsubnet.yaml)

### Step 3:

    (tmos)# create net self 10.128.4.60/14 allow-service all vlan openshift_vxlan

Subnet from the **f5-server** hostsubnet create above. Used .60 to be consistent with Big-IP internal interface

![diagram](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/openshift-4-7/standalone/diagram/2021-06-30_14-38-24.png)

## Create a new partition on your BIG-IP system

### Step 4:

    (tmos)# create auth partition OpenShift

This needs to match the partition in the controller configuration created by the CIS Operator

## Installing the F5 Container Ingress Services Operator in OpenShift

In OpenShift, CIS can be installed manually using a a yaml deployment manifest or using the Operator in OpenShift. The CIS Operator is a packaged deployment of CIS and will use Helm Charts to create the deployment. This user-guide provide additional information and examples when using the CIS Operator in OpenShift

### Step 5:

Create BIG-IP login credentials for use with Operator Helm charts

    oc create secret generic bigip-login  -n kube-system --from-literal=username=admin  --from-literal=password=<secret>

### Step 6:

Locate the F5 Container Ingress Services Operator in OpenShift OperatorHub as shown in the diagram below. Recommend search for F5 

![diagram](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/operator/diagrams/2021-06-10_12-59-30.png)

Select the Operator to Install. In this example I am installing the latest Operator 1.7.0. Select the Install tab as shown in the diagram

![diagram](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/operator/diagrams/2021-06-10_13-20-27.png)

### Step 7:

Install the Operator and provide the installation mode, installed namespaces and approval strategy. In this user-guide and demo I am using the defaults

![diagram](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/operator/diagrams/2021-06-10_13-47-45.png)

Operator will take a few minutes to install

![diagram](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/operator/diagrams/2021-06-10_13-50-10.png)

Once installed select the View Operator tab

![diagram](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/operator/diagrams/2021-06-10_13-51-02.png)

Now that the operator is installed you can create an instance of CIS. This will deploy CIS in OpenShift

![diagram](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/operator/diagrams/2021-06-14_14-07-36.png)

### Step 8:

Note that currently some fields may not be represented in form so its best to use the "YAML View" for full control of object creation. Select the "YAML View"

![diagram](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/operator/diagrams/2021-06-14_14-14-41.png)

Enter requirement objects in the YAML View. Please add the recommended setting below:

* Remove **agent as3** as this is default
* Change repo image to **f5networks/cntr-ingress-svcs**. By default OpenShift will pull the image from Docker. 
* Change the user to **registry.connect.redhat.com** so OpenShift will be pull the published image from the RedHat Ecosystem Catalog [repo](https://catalog.redhat.com/software/containers/f5networks/cntr-ingress-svcs/5ec7ad05ecb5246c0903f4cf)

```
apiVersion: cis.f5.com/v1
kind: F5BigIpCtlr
metadata:
  name: f5-server
  namespace: openshift-operators
spec:
  args:
    log_as3_response: true
    manage_routes: true
    log_level: DEBUG
    route_vserver_addr: 10.192.125.65
    bigip_partition: OpenShift
    openshift_sdn_name: /Common/openshift_vxlan
    bigip_url: 10.192.125.60
    insecure: true
    pool-member-type: cluster
  bigip_login_secret: bigip-login
  image:
    pullPolicy: Always
    repo: f5networks/cntr-ingress-svcs
    user: registry.connect.redhat.com
  namespace: kube-system
  rbac:
    create: true
  resources: {}
  serviceAccount:
    create: true
  version: latest
```

Select the Create tab

### Step 9:

Validate CIS deployment. Select Workloads/Deployments 

![diagram](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/operator/diagrams/2021-06-14_14-42-54.png)

Select the **f5-bigip-ctlr-operator** to see more details on the CIS deployment. Also validate the CIS deployment image

![diagram](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/operator/diagrams/2021-06-14_14-45-08.png)

## Installing the Demo App in OpenShift

### Step 10:

Deploy demo app in OpenShift. This could be done using the OpenShift UI or CLI. In this guide i use the CLI. Demo app repo available below 

```
# oc create -f demo-app/
deployment.apps/f5-demo created
service/f5-demo created
```
demo-app [repo](https://github.com/mdditt2000/k8s-bigip-ctlr/tree/main/user_guides/openshift-4-7/standalone/demo-app)

You can validate the demo app install via the OpenShift UI

![diagram](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/openshift-4-7/standalone/diagram/2021-06-30_11-39-52.png)

## Create Route for Ingress traffic to Demo App

### Step 11:

Create basic route for Ingress traffic from BIG-IP to Demo App 

```
# oc create -f f5-demo-route-basic.yaml
route.route.openshift.io/f5-demo-route-basic created
```

f5-demo-route-basic [repo](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/openshift-4-7/standalone/route/f5-demo-route-basic.yaml)

Validate the route via the OpenShift UI under the Networking/Routes

![diagram](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/openshift-4-7/standalone/diagram/2021-06-30_13-59-43.png)

Validate the route via the BIG-IP

![diagram](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/openshift-4-7/standalone/diagram/2021-06-30_14-00-53.png)