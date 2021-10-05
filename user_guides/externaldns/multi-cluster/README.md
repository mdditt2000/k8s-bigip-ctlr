# ExternalDNS for Multi-Site Kubernetes using F5 CIS with BIG-IP LTM and DNS

ExternalDNS allows user to control DNS records dynamically via Kubernetes CRD resources in a DNS provider-agnostic way. This user-guide documents using F5 CIS with BIG-IP LTM and DNS in a multi-site kubernetes deployment. 

Looking at the diagram below there are two Data Centers **east** and **west**. Both Data Center have standalone BIG-IP LTM and DNS. The BIG-IP DNS devices are synchronized, to share Data Center, Server, Wide IP, and Virtual Server availability. Each Data Centers has Kubernetes deployed with duplicate applications. However Data Center **west** has a newer version of Kubernetes installed. Creating a second cluster with traffic distribution helps validation of newer kubernetes version, high availability, scaling etc. F5 CIS with BIG-IP LTM and DNS using ExternalDNS solves the multi-site challenges.

Demo on YouTube [video]()

![architecture](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/externaldns/multi-cluster/diagrams/2021-09-27_12-21-58.png)

## Prerequisites

* Recommend AS3 version 3.30 [repo](https://github.com/F5Networks/f5-appsvcs-extension/releases/tag/v3.30.0)
* CIS 2.6 [repo](coming)
* Clouddocs [documentation](https://clouddocs.f5.com/containers/latest/userguide/crd/externaldns.html)
* Global DNS provisioned and synchronized between Data Centers

## Step 1: Deploy CIS for both Data Centers

CIS 2.6 communicates directly with BIG-IP DNS via the Rest API and requires gtm-bigip-username and password. Since BIG-IP LTM and DNS are on the **same BIG-IP** you can re-use the secret generic bigip-login when deploying CIS as shown below.

Add the following parameters to THE CIS deployment

* --custom-resource-mode=true - Configure CIS to only monitor CRDs. CIS will ignore all other resources
* --gtm-bigip-username - Provide username for CIS to access GTM
* --gtm-bigip-password - Provide password for CIS to access GTM
* --gtm-bigip-url - Provide url for CIS to access GTM. CIS uses the python SDK to configure GTM 

Cluster **east**

```
- args: 
    - "--gtm-bigip-username=$(BIGIP_USERNAME)"
    - "--gtm-bigip-password=$(BIGIP_PASSWORD)"
    - "--gtm-bigip-url=192.168.200.91"
    - "--bigip-username=$(BIGIP_USERNAME)"
    - "--bigip-password=$(BIGIP_PASSWORD)"
    - "--bigip-url=192.168.200.91"
    - "--bigip-partition=k8s"
    - "--pool-member-type=cluster"
    - "--flannel-name=fl-vxlan"
    - "--log-level=DEBUG"
    - "--insecure=true"
    - "--custom-resource-mode=true"
    - "--as3-validation=true"
    - "--log-as3-response=true"
```

Cluster **west**

```
- args: 
    - "--gtm-bigip-username=$(BIGIP_USERNAME)"
    - "--gtm-bigip-password=$(BIGIP_PASSWORD)"
    - "--gtm-bigip-url=192.168.200.60"
    - "--bigip-username=$(BIGIP_USERNAME)"
    - "--bigip-password=$(BIGIP_PASSWORD)"
    - "--bigip-url=192.168.200.60"
    - "--bigip-partition=k8s"
    - "--pool-member-type=cluster"
    - "--flannel-name=fl-vxlan"
    - "--log-level=DEBUG"
    - "--insecure=true"
    - "--custom-resource-mode=true"
    - "--as3-validation=true"
    - "--log-as3-response=true"
```

Deploy CIS in both **east** and **west**

```
kubectl create secret generic bigip-login -n kube-system --from-literal=username=admin --from-literal=password=<secret>
kubectl create serviceaccount k8s-bigip-ctlr -n kube-system
kubectl create clusterrolebinding k8s-bigip-ctlr-clusteradmin --clusterrole=cluster-admin --serviceaccount=kube-system:k8s-bigip-ctlr
kubectl create -f f5-cluster-deployment.yaml
kubectl create -f f5-bigip-node.yaml
```
**Note** f5-bigip-node is required for Flannel

* cis-deployment **east** [repo](https://github.com/mdditt2000/k8s-bigip-ctlr/tree/main/user_guides/externaldns/multi-cluster/east/cis-deployment)
* cis-deployment **west** [repo](https://github.com/mdditt2000/k8s-bigip-ctlr/tree/main/user_guides/externaldns/multi-cluster/west/cis-deployment)

## Step 2: Deploy F5 Demo App for both Data Centers

Deploy the test F5 demo deployment and service. This is a simple application on port 80 and requires a Host Header

```
kubectl create -f pod-deployment
```

* pod-deployment **east** [repo](https://github.com/mdditt2000/k8s-bigip-ctlr/tree/main/user_guides/externaldns/multi-cluster/east/pod-deployment)
* pod-deployment **west** [repo](https://github.com/mdditt2000/k8s-bigip-ctlr/tree/main/user_guides/externaldns/multi-cluster/west/pod-deployment)

## Step 3: Create the VirtualServers for both Data Centers

**Note** CIS requires **east** amd **west** **Data Center** created on both BIG-IP DNS devices

* **DataCenter** **east** for **Kubernetes v19** and **Kubernetes v20 clusters**

![DataCenter](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/externaldns/multi-cluster/diagrams/2021-10-04_20-45-10.png)

* **DataCenter** **west** for **Kubernetes v19** and **Kubernetes v20 clusters**

![DataCenter](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/externaldns/multi-cluster/diagrams/2021-10-04_20-46-35.png)

Example of **DataCenter** for both **east** amd **west**

![DataCenter](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/externaldns/multi-cluster/diagrams/2021-10-04_20-54-11.png)

**Note** CIS requires the **Server** for **east** amd **west** on both BIG-IP DNS devices

* **Servers** for **east** BIG-IP DNS under GSLB(DNS) by referring:

    - **DataCenter** with **BIG-IP device**

![Servers](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/externaldns/multi-cluster/diagrams/2021-10-04_20-38-58.png)

* **Servers** under GSLB(DNS) by referring:

    - **External SelfIP**

![Servers](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/externaldns/multi-cluster/diagrams/2021-10-04_21-00-01.png)

* **Servers** under GSLB(DNS) by referring:

    - **Virtual Server Discovery enabled**

![Servers](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/externaldns/multi-cluster/diagrams/2021-10-04_21-00-39.png)
    
**Note** Virtual Server Discovery must be enabled for this solution to work. We plan to enhance this in a upcoming release of CIS

* **Servers** for **west** BIG-IP DNS under GSLB(DNS) by referring:

    - **DataCenter** with **BIG-IP device**

![Servers](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/externaldns/multi-cluster/diagrams/2021-10-04_21-04-25.png)

* **Servers** under GSLB(DNS) by referring:

    - **External SelfIP**

![Servers](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/externaldns/multi-cluster/diagrams/2021-10-04_21-05-00.png)

* **Servers** under GSLB(DNS) by referring:

    - **Virtual Server Discovery enabled**

![Servers](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/externaldns/multi-cluster/diagrams/2021-10-04_21-05-47.png)
    
**Note** Virtual Server Discovery must be enabled for this solution to work. We plan to enhance this in a upcoming release of CIS

Create the mysite and myapp virtualservers CRDs for both for both Data Centers

```
kubectl create -f vs-myapp.yaml
kubectl create -f vs-mysite.yaml
kubectl create -f customresourcedefinitions.yml
```
* crd-resources **east** [repo](https://github.com/mdditt2000/k8s-bigip-ctlr/tree/main/user_guides/externaldns/multi-cluster/east/crd-example)
* crd-resources **west** [repo](https://github.com/mdditt2000/k8s-bigip-ctlr/tree/main/user_guides/externaldns/multi-cluster/west/crd-example)

Verify DataCenter and Server list could learn the new virtualservers LTM in the serverlist for **east**

![virtualservers](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/externaldns/multi-cluster/diagrams/2021-10-04_21-14-19.png)

Verify DataCenter and Server list could learn the new virtualservers LTM in the serverlist for **west**

![virtualservers](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/externaldns/multi-cluster/diagrams/2021-10-04_21-14-55.png)

If all the **virtualservers** are created, **green** and synchronized you can continue to **Step 4** and create the Wide IPs for both **east** and **west**

## Step 4: Create the WideIP's using the ExternalDNS CRDs for both east and west

The diagram below show the **VirtualServer** and **ExternalDNS CRD** for **mysite.f5demo.com** in **east**. Important to **Note** the following:

* **Host:** **mysite.f5demo.com** in the **VirtualServer** CRD needs to match **domainName: mysite.f5demo.com**. Pool needs to watch the Datacenter **pools name: east.mysite.f5demo.com** **ExternalDNS CRD**
* Use the following string for the **GSLB monitor** in the ExternalDNS CRD

```
 monitor:
      type: http
      send: "GET / HTTP/1.1\r\nHost: mysite.f5demo.com\r\n"
```

* **dataServerName: /Common/east in the ExternalDNS CRD needs to match DataCenter Server name

![mysite](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/externaldns/multi-cluster/diagrams/2021-10-04_21-34-30.png)

Create the mysite and myapp EDNS CRDs for both **east** and **west*

```
kubectl create -f edns-myapp.yaml
kubectl create -f edns-mysite.yaml
```

* crd-resources **east** [repo](https://github.com/mdditt2000/k8s-bigip-ctlr/tree/main/user_guides/externaldns/multi-cluster/east/crd-example)
* crd-resources **west** [repo](https://github.com/mdditt2000/k8s-bigip-ctlr/tree/main/user_guides/externaldns/multi-cluster/west/crd-example)

Validate the WIDE IP list. You should see both Wide IP created for mysite and myapp and both green status

![architecture](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/externaldns/multi-cluster/diagrams/2021-10-04_21-44-28.png)

Validate the pool members for the WIDE IP list. You should see two members in the pool as shown in the diagram below

![architecture](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/externaldns/multi-cluster/diagrams/2021-10-04_21-49-56.png)

## Step 5: Validating multi-site DNS availability

**note** BIG-IP needs to listeners for both UDP and TCP for both **east**

![architecture](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/externaldns/multi-cluster/diagrams/2021-10-04_22-08-45.png)

**note** BIG-IP needs to listeners for both UDP and TCP for both **west**

![architecture](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/externaldns/multi-cluster/diagrams/2021-10-04_22-11-50.png)

**note** Setup the correct dedication for the NS records to point to the BIG-IP DNS. NS record for **mysite.f5demo.com** and **myapp.f5demo.com** are **east** and **west** BIG-IP DNS devices

![delegation](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/externaldns/multi-cluster/diagrams/2021-10-04_22-20-36.png)

Validate connectively to **mysite.f5demo.com** 

![delegation](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/externaldns/multi-cluster/diagrams/2021-10-04_22-28-35.png)

Validate connectively to **myapp.f5demo.com** 

![delegation](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/externaldns/multi-cluster/diagrams/2021-10-04_22-28-17.png)