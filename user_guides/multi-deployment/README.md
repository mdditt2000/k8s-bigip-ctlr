# Platform-agnostic Container Environments using DNS Failover

Today, organizations are increasingly deploying multiple container environment. Deploying multiple environment can improve availability, isolation and scalability. This user-guide demonstrates how F5 Container Ingress Services (CIS) can automate BIP-IP to provide Ingress services for a multiple platform-agnostic container environments.

## OpenShift and Kubernetes Container Environments

In this user-guide, we have deployed an OpenShift and Kubernetes container environments running identical applications. BIG-IP is platform-agnostic, using DNS to distribute traffic between the two container environments. This simple but powerful approach enables users the flexibility to complete an container environment proof of concept or migrating applications between environments. Since CIS uses the Kubernetes API the resource definitions for OpenShift and Kubernetes are identical except for the public IPs. Diagram below represents the OpenShift and Kubernetes environments.

![architecture](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/platform-agnostic/diagram/2022-02-28_09-39-06.png)

Demo on YouTube [video]()

This user-guide demonstrates an application having a Wide IP's HOST name **cafe.example.com**  which answers using **round-robin** for the OpenShift or Kubernetes environments. DNS has no layer seven path awareness and therefore DNS monitors are required to determine the health of the applications **/coffee** and **/tea**. Each ExternalDNS CRD specifies the DNS monitors on BIG-IP. Recommended to work with your F5 Solution Architect to discuss DNS monitoring and scaling. If a monitor detects the http status failure, the Wide IP is removed from the DNS query.

### Environment parameters

* BIG-IP LTM and DNS configured on the same device
* Configure BIG-IP DNS iQuery so that BIG-IP systems can communicate with each other for datacenter **ocp** and **k8s**

![iQuery](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/platform-agnostic/diagram/2022-02-28_11-11-06.png)

## OpenShift Container Environments

### Step 1: Deploy CIS

In this user-guide, CIS is deployed using a manifest. CIS can also be deployed using the Operator from the OpenShift dashboard. Follow user-guide to deploy CIS using the [Operator CIS](https://github.com/mdditt2000/k8s-bigip-ctlr/tree/main/user_guides/operator#readme)

Add the following parameters to the CIS deployment

* --custom-resource-mode=true - Configure CIS to watch for CRDs. ExternalDNS is not currently supported using OpenShift Routes
* --bigip-partition=ocp - CIS uses BIG-IP tenant OpenShift to manage CRDs
* --openshift-sdn-name=/Common/openshift_vxlan - CNI policy on BIG-IP to connect to the PODs in OpenShift

```
args: [
  # See the k8s-bigip-ctlr documentation for information about
  # all config options
  # https://clouddocs.f5.com/containers/latest/
    "--bigip-username=$(BIGIP_USERNAME)",
    "--bigip-password=$(BIGIP_PASSWORD)",
    "--bigip-url=10.192.125.60",
    "--bigip-partition=ocp",
    "--gtm-bigip-username=$(BIGIP_USERNAME)",
    "--gtm-bigip-password=$(BIGIP_PASSWORD)",
    "--gtm-bigip-url=10.192.125.60",
    "--namespace=default",
    "--pool-member-type=cluster",
    "--openshift-sdn-name=/Common/openshift_vxlan",
    "--log-level=INFO",
    "--insecure=true",
    "--custom-resource-mode=true",
    "--as3-validation=true",
    "--log-as3-response=true",
  ]
```

#### Deploy CIS in OpenShift

```
oc create secret generic bigip-login -n kube-system --from-literal=username=admin --from-literal=password=<secret>
oc create -f bigip-ctlr-clusterrole.yaml
oc create -f f5-bigip-ctlr-deployment.yaml
```

cis-deployment [repo](https://github.com/mdditt2000/k8s-bigip-ctlr/tree/main/user_guides/platform-agnostic/ocp/cis/cis-deployment)

**Note** Do not forget the OpenShift-SDN CNI [repo](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/platform-agnostic/ocp/cni/f5-openshift-hostsubnet-01.yaml)

## Kubernetes Container Environments

### Step 2: Deploy CIS

The biggest benefit for using CRDs is their no limitations on how many Public IPs **Virtual Server** create on BIG-IP. However to maintain similarity with Routes we using HOST Header Load balancing to determine the backend application. In this example the backend is **/tea,/coffee and /mocha** with the same Public IP address 10.192.125.65

Add the following parameters to THE CIS deployment

* --custom-resource-mode=true - Configure CIS to watch for CRDs
* --bigip-partition=k8s - CIS uses BIG-IP tenant Kubernetes to manage CRDs
* --flannel-name=fl-vxlan - CNI policy on BIG-IP to connect to the PODs in Kubernetes

```
args: [
  # See the k8s-bigip-ctlr documentation for information about
  # all config options
  # https://clouddocs.f5.com/containers/latest/
    "--bigip-username=$(BIGIP_USERNAME)",
    "--bigip-password=$(BIGIP_PASSWORD)",
    "--bigip-url=192.168.200.60",
    "--bigip-partition=k8s",
    "--gtm-bigip-username=$(BIGIP_USERNAME)",
    "--gtm-bigip-password=$(BIGIP_PASSWORD)",
    "--gtm-bigip-url=192.168.200.60",
    "--namespace=default",
    "--pool-member-type=cluster",
    "--flannel-name=fl-vxlan",
    "--log-level=INFO",
    "--insecure=true",
    "--custom-resource-mode=true",
    "--as3-validation=true",
    "--log-as3-response=true",
]
```

#### Deploy CIS in Kubernetes

```
kubectl create secret generic bigip-login -n kube-system --from-literal=username=admin --from-literal=password=<secret>
kubectl create -f bigip-ctlr-clusterrole.yaml
kubectl create -f f5-bigip-ctlr-deployment.yaml
```

cis-deployment [repo](https://github.com/mdditt2000/k8s-bigip-ctlr/tree/main/user_guides/platform-agnostic/k8s/cis/cis-deployment)

**Note** Do not forget the Flannel CNI [repo](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/platform-agnostic/k8s/cni/f5-bigip-node.yaml)

## Creating Custom Resource Definitions 

### Step 3: Creating VirtualServer and ExternalDNS CRDs using OpenShift

Use-case for the CRDs:

- unsecure
- Health monitor of the backend application using HOST **cafe.example.com** and **PATH /coffee, and /tea**
- Same Hostname **cafe.example.com** for OpenShift and Kubernetes environments

Diagram below displays the example of **vs-tea** with the **edns-cafe** for the following use-case

![crd-ocp](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/platform-agnostic/diagram/2022-02-28_16-44-15.png)

#### Create CRDs Schema in OpenShift

**Note:** CIS requires the CustomResourceDefinition schema

```
oc create -f CustomResourceDefinition.yaml
```

CRD Schema [repo](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/platform-agnostic/ocp/cis/cafe/cis-crd-schema/customresourcedefinitions.yml)

#### Create VirtualServer and ExternalDNS CRDs in OpenShift

```
oc create -f vs-tea.yaml
oc create -f vs-coffee.yaml
oc create -f edns-cafe.yaml
```

CRD [repo](https://github.com/mdditt2000/k8s-bigip-ctlr/tree/main/user_guides/platform-agnostic/ocp/cis/cafe/unsecure)

#### Validate CRD

**Note** Sadly OpenShift does not have the same Dashboard for CRDs. Therefore you need to use the OpenShift CLI

```
# oc get crd,vs,externaldns -n default
NAME                                 HOST               TLSPROFILENAME   HTTPTRAFFIC   IPADDRESS        IPAMLABEL   IPAMVSADDRESS   STATUS   AGE
virtualserver.cis.f5.com/vs-coffee   cafe.example.com                                  10.192.125.121               None            Ok       2d23h
virtualserver.cis.f5.com/vs-tea      cafe.example.com                                  10.192.125.121               None            Ok       2d23h

NAME                               DOMAINNAME         AGE     CREATED ON
externaldns.cis.f5.com/edns-cafe   cafe.example.com   2d23h   2022-02-26T01:35:19Z
```

#### Validate CRD using the BIG-IP

![big-ip CRD](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/platform-agnostic/diagram/2022-02-28_16-52-50.png)

#### Validate CRD policy for cafe.example.com on BIG-IP

![big-ip pools](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/platform-agnostic/diagram/2022-02-28_16-52-04.png)

### Step 4: Creating VirtualServer and ExternalDNS CRDs using Kubernetes

Use-case for the CRDs:

- unsecure
- Health monitor of the backend application using HOST **cafe.example.com** and **PATH /coffee, and /tea**
- Same Hostname **cafe.example.com** for OpenShift and Kubernetes environments

Diagram below displays the example of **vs-tea** with the **edns-cafe** for the following use-case

![crd-k8s](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/platform-agnostic/diagram/2022-02-28_16-45-08.png)

#### Create CRDs Schema in Kubernetes

**Note:** CIS requires the CustomResourceDefinition schema

```
kubectl create -f CustomResourceDefinition.yaml
```

CRD Schema [repo](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/platform-agnostic/k8s/cis/cafe/cis-crd-schema/customresourcedefinitions.yml)

#### Create VirtualServer and ExternalDNS CRDs in Kubernetes

```
kubectl create -f vs-tea.yaml
kubectl create -f vs-coffee.yaml
kubectl create -f edns-cafe.yaml
```

CRD [repo](https://github.com/mdditt2000/k8s-bigip-ctlr/tree/main/user_guides/platform-agnostic/k8s/cis/cafe/unsecure)

#### Validate CRD

```
# kubectl get crd,vs,externaldns -n default
NAME                                 HOST               TLSPROFILENAME   HTTPTRAFFIC   IPADDRESS       IPAMLABEL   IPAMVSADDRESS   STATUS   AGE
virtualserver.cis.f5.com/vs-coffee   cafe.example.com                                  10.192.75.121               None            Ok       3d4h
virtualserver.cis.f5.com/vs-tea      cafe.example.com                                  10.192.75.121               None            Ok       3d4h

NAME                               DOMAINNAME         AGE     CREATED ON
externaldns.cis.f5.com/edns-cafe   cafe.example.com   2d23h   2022-02-26T01:35:43Z
```

#### Validate CRD using the BIG-IP

![big-ip CRD](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/platform-agnostic/diagram/2022-02-28_16-59-13.png)

Validate Kubernetes CRD pool-members using the BIG-IP

![big-ip pools](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/platform-agnostic/diagram/2022-02-28_16-59-41.png)

### Step 5: Validate the BIG-IP Wide IPs and DNS Failover

#### Validate the BIG-IP GSLB Wide IP

![wide-ip](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/platform-agnostic/diagram/2022-03-01_10-13-48.png)

#### Validate the BIG-IP GLSB Pools

Each pool represents a container environment. In this user-guide we have a **ocp** and **k8s** pools

![wide-ip](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/platform-agnostic/diagram/2022-03-01_10-14-21.png)

#### Validate the **ocp** Data Center GLSB Pool on BIG-IP

![wide-ip](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/platform-agnostic/diagram/2022-03-01_10-17-31.png)

**Note** availability shows green. Monitors are able to successfully complete the health checks

#### Validate the **ocp** Data Center GLSB Pool answer on BIG-IP

This is the public IP returned by the BIG-IP DNS to the clients DNS query

![wide-ip](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/platform-agnostic/diagram/2022-03-01_10-17-59.png)

#### Validate the **k8s** Data Center GLSB Pool on BIG-IP

![wide-ip](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/platform-agnostic/diagram/2022-03-01_10-18-28.png)

**Note** availability shows green. Monitors are able to successfully complete the health checks

#### Validate the **k8s** Data Center GLSB Pool answer on BIG-IP

This is the public IP returned by the BIG-IP DNS to the clients DNS query

![wide-ip](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/platform-agnostic/diagram/2022-03-01_10-18-57.png)

#### Replica the **k8s** pods to zero

Replica the application pods in the Kubernetes container environment to zero. BIG-IP DNS will detect the replica and remove the **k8s** Data Center from the WIDE IP. Only **ocp** will be available

```
❯ kubectl scale --current-replicas=3 --replicas=0 deployment/coffee
deployment.apps/coffee scaled
❯ kubectl scale --current-replicas=3 --replicas=0 deployment/tea
deployment.apps/tea scaled
```

![wide-ip](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/platform-agnostic/diagram/2022-03-01_10-52-48.png)

#### Connect to the Public IP

**Note** All traffic will connect to the **ocp** cluster. I can verify this by the server address

![wide-ip](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/platform-agnostic/diagram/2022-03-01_10-55-16.png)