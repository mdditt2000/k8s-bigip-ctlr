# OpenShift 4.7 and F5 Container Ingress Services (CIS) User-Guide for BIG-IP Cluster

This user guide is created to document OpenShift 4.7 integration with a BIG-IP cluster using CIS. This user guide provides the configuration for a BIG-IP cluster using **OpenShift SDN**. BIG-IP cluster is pre-configured using **manual with incremental sync**

![diagram](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/openshift-4-7/cluster/diagram/2021-07-06_13-18-55.png)

Demo on YouTube [video]()

## Environment parameters

* OpenShift 4.7 on vSphere with installer-provisioned infrastructure
* CIS 2.5
* AS3: 3.29
* BIG-IP 16.0.1.1 cluster

## Prerequisite

CIS uses the AS3 declarative API. AS3 extension installation on BIG-IP is required. Follow the link to install AS3
 
* Install AS3 on BIG-IP
https://clouddocs.f5.com/products/extensions/f5-appsvcs-extension/latest/userguide/installation.html

### Step 1: Create the VXLAN profile and tunnels

On bigip-01 create the VXLAN profile and tunnel

```
(tmos)# create net tunnels vxlan vxlan-mp flooding-type multipoint
(tmos)# create net tunnels tunnel openshift_vxlan key 0 profile vxlan-mp local-address 10.192.125.62 secondary-address 10.192.125.60 traffic-group traffic-group-1
```
![diagram](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/openshift-4-7/cluster/diagram/2021-07-06_13-07-43.png)

On bigip-02 create the VXLAN tunnel

```
(tmos)# create net tunnels tunnel openshift_vxlan key 0 profile vxlan-mp local-address 10.192.125.62 secondary-address 10.192.125.61 traffic-group traffic-group-1
```
![diagram](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/openshift-4-7/cluster/diagram/2021-07-06_13-08-24.png)

### Step 2: Create a new OpenShift HostSubnets

Create the hostsubnets for the BIP-IP. This will provide the subnet for creating the tunnel self-IP

    oc create -f f5-openshift-hostsubnet-01.yaml
    oc create -f f5-openshift-hostsubnet-02.yaml
    oc create -f f5-openshift-hostsubnet-float.yaml

```
# oc get hostsubnets
NAME                        HOST                        HOST IP         SUBNET          EGRESS CIDRS   EGRESS IPS
f5-server-01                f5-server-01                10.192.125.60   10.129.4.0/23
f5-server-02                f5-server-02                10.192.125.61   10.130.4.0/23
f5-server-float             f5-server-float             10.192.125.62   10.131.4.0/23
ocp-pm-bwmmz-master-0       ocp-pm-bwmmz-master-0       10.192.75.229   10.130.0.0/23
ocp-pm-bwmmz-master-1       ocp-pm-bwmmz-master-1       10.192.75.231   10.129.0.0/23
ocp-pm-bwmmz-master-2       ocp-pm-bwmmz-master-2       10.192.75.230   10.128.0.0/23
ocp-pm-bwmmz-worker-9ch4b   ocp-pm-bwmmz-worker-9ch4b   10.192.75.234   10.129.2.0/23
ocp-pm-bwmmz-worker-lws6s   ocp-pm-bwmmz-worker-lws6s   10.192.75.235   10.131.0.0/23
ocp-pm-bwmmz-worker-qdhgx   ocp-pm-bwmmz-worker-qdhgx   10.192.75.233   10.128.2.0/23
```

f5-openshift-hostsubnet.yaml [repo](https://github.com/mdditt2000/k8s-bigip-ctlr/tree/main/user_guides/openshift-4-7/cluster/cis)

### Step 3: Create the self IPs for the VXLAN CNI

Create the self IP address for the VXLAN CNI on each BIG-IP. The subnet mask you assign to the self IP must match the one that the OpenShift SDN assigns to nodes. **Note** that is a /14 by default. Be sure to specify a floating traffic group (for example, traffic-group-1). Otherwise, the self IP will use the BIG-IP system’s default

On bigip-01 create the self IP from hostsubnets **f5-server-01**
```
(tmos)# create net self 10.129.4.60/14 allow-service all vlan openshift_vxlan
```
![diagram](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/openshift-4-7/cluster/diagram/2021-07-06_13-49-15.png)

On bigip-02 create the self IP from hostsubnets **f5-server-02**
```
(tmos)# create net self 10.130.4.61/14 allow-service all vlan openshift_vxlan
```
![diagram](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/openshift-4-7/cluster/diagram/2021-07-06_13-50-21.png)

On the active BIG-IP, create a floating IP address in the subnet assigned by the OpenShift SDN from from hostsubnets **f5-server-float**
```
(tmos)# create net self 10.131.4.62/14 allow-service default traffic-group traffic-group-1 vlan openshift_vxlan
```
![diagram](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/openshift-4-7/cluster/diagram/2021-07-06_14-12-10.png)

### Step 4: Create a new partition on your BIG-IP system

    (tmos)# create auth partition OpenShift

This needs to match the partition in the controller configuration created by the CIS Operator

## Installing the F5 Container Ingress Services OpenShift

In OpenShift, CIS can be installed manually using a a yaml deployment manifest or using the Operator in OpenShift. The CIS Operator is a packaged deployment of CIS and will use Helm Charts to create the deployment. This user-guide provide will be creating CIS using a yaml deployment manifest and not the CIS Operator due to GitHub [issue](https://github.com/F5Networks/k8s-bigip-ctlr/issues/1813)

### Step 5: Create CIS Controller, BIG-IP credentials and RBAC Authentication

Create both bigip-01 and bigip-02 yaml deployment manifests

```
# oc create secret generic bigip-login --namespace kube-system --from-literal=username=admin --from-literal=password=<secret>
# oc create serviceaccount bigip-ctlr -n kube-system
# oc create -f f5-openshift-clusterrole.yaml
# oc create -f f5-bigip-01-deployment.yaml
# oc create -f f5-bigip-02-deployment.yaml
# oc adm policy add-cluster-role-to-user cluster-admin -z bigip-ctlr -n kube-system
```

### Step 6: Validate CIS deployment. Select Workloads/Deployments

![diagram](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/openshift-4-7/cluster/diagram/2021-07-07_15-30-56.png)

### Step 10: Installing the Demo App in OpenShift

Deploy demo app in OpenShift. This could be done using the OpenShift UI or CLI. In this guide i use the CLI. Demo app repo available below 

```
# oc create -f demo-app/
deployment.apps/f5-demo created
service/f5-demo created
```
demo-app [repo](https://github.com/mdditt2000/k8s-bigip-ctlr/tree/main/user_guides/openshift-4-7/cluster/demo-app)

You can validate the demo app install via the OpenShift UI

![diagram](https://github.com/mdditt2000/openshift-4-7/blob/master/standalone/diagram/2021-06-30_11-39-52.png)

## Create Route for Ingress traffic to Demo App

### Step 11:

Create basic route for Ingress traffic from BIG-IP to Demo App 

```
# oc create -f f5-demo-route-basic.yaml
route.route.openshift.io/f5-demo-route-basic created
```

f5-demo-route-basic [repo](https://github.com/mdditt2000/k8s-bigip-ctlr/tree/main/user_guides/openshift-4-7/cluster/route)

Validate the route via the OpenShift UI under the Networking/Routes

![diagram](https://github.com/mdditt2000/openshift-4-7/blob/master/standalone/diagram/2021-06-30_13-59-43.png)

Validate the route via the BIG-IP

![diagram](https://github.com/mdditt2000/openshift-4-7/blob/master/standalone/diagram/2021-06-30_14-00-53.png)