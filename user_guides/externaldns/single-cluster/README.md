# ExternalDNS for Kubernetes using F5 CIS with BIG-IP LTM and DNS

ExternalDNS allows user to control DNS records dynamically via Kubernetes CRD resources in a DNS provider-agnostic way. This user-guide documents using F5 CIS with BIG-IP LTM and DNS on the same device for a single cluster as shown in the diagram

Demo on YouTube [video]()

![architecture](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/externaldns/single-cluster/diagram/2021-09-17_10-24-26.png)

## Prerequisites

* Recommend AS3 version 3.30 [repo](https://github.com/F5Networks/f5-appsvcs-extension/releases/tag/v3.30.0)
* CIS 2.6 [repo](coming)
* Clouddocs [documentation](https://clouddocs.f5.com/containers/latest/userguide/crd/externaldns.html)
* Global DNS license under Resource Provisioning Section

## Step 1: Deploy CIS

CIS 2.6 communicates directly with BIG-IP DNS via the Rest API and requires gtm-bigip-username and password. Since BIG-IP LTM and DNS are on the same device you can re-use the secret generic bigip-login when deploying CIS as shown below.

Add the following parameters to THE CIS deployment

* --custom-resource-mode=true - Configure CIS to only monitor CRDs. CIS will ignore all other resources
* --gtm-bigip-username - Provide username for CIS to access GTM
* --gtm-bigip-password - Provide password for CIS to access GTM
* --gtm-bigip-url - Provide url for CIS to access GTM. CIS uses the python SDK to configure GTM 

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

Deploy CIS in both locations

```
kubectl create secret generic bigip-login -n kube-system --from-literal=username=admin --from-literal=password=<secret>
kubectl create serviceaccount k8s-bigip-ctlr -n kube-system
kubectl create clusterrolebinding k8s-bigip-ctlr-clusteradmin --clusterrole=cluster-admin --serviceaccount=kube-system:k8s-bigip-ctlr
kubectl create -f f5-cluster-deployment.yaml
kubectl create -f f5-bigip-node.yaml
```
**Note** f5-bigip-node is required for Flannel

cis-deployment [repo](https://github.com/mdditt2000/k8s-bigip-ctlr/tree/main/user_guides/externaldns/single-cluster/cis-deployment)

## Step 2: Deploy F5 Demo App 

Deploy the test F5 demo deployment and service. This is a simple application on port 80 and requires a Host Header

```
kubectl create -f pod-deployment
```

pod-deployment [repo](https://github.com/mdditt2000/k8s-bigip-ctlr/tree/main/user_guides/externaldns/single-cluster/pod-deployment)

## Step 3: Create the VirtualServers

**Note** CIS requires the following prerequisites created on BIG-IP DNS

* **DataCenter** using the default options

![DataCenter](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/externaldns/single-cluster/diagram/2021-09-17_10-49-20.png)

* **Servers** under GSLB(DNS) by referring:

    - **DataCenter** with **BIG-IP device**

![Servers](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/externaldns/single-cluster/diagram/2021-09-20_14-17-02.png)

* **Servers** under GSLB(DNS) by referring:

    - **External SelfIP**

![Servers](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/externaldns/single-cluster/diagram/2021-09-20_14-17-58.png)

* **Servers** under GSLB(DNS) by referring:

    - **Virtual Server Discovery enabled**

![Servers](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/externaldns/single-cluster/diagram/2021-09-20_14-18-23.png)
    
**Note** Virtual Server Discovery must be enabled for this solution to work. We plan to enhance this in a upcoming release of CIS

Create the mysite and myapp virtualservers CRDs

```
kubectl create -f vs-myapp.yaml
kubectl create -f vs-mysite.yaml
kubectl create -f customresourcedefinitions.yml
```
crd-resources [repo](https://github.com/mdditt2000/k8s-bigip-ctlr/tree/main/user_guides/externaldns/single-cluster/crd-example)

Validate both **virtualservers** crd's are created

![virtualservers](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/externaldns/single-cluster/diagram/2021-09-17_13-39-20.png)

Connect the **mysite.f5demo.com**

![mysite](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/externaldns/single-cluster/diagram/2021-09-17_13-40-14.png)

Connect the **myapp.f5demo.com**

![myapp](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/externaldns/single-cluster/diagram/2021-09-17_13-39-58.png)

Verify DataCenter and Server list could learn the new virtualservers LTM in the serverlist

![serverlist](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/externaldns/single-cluster/diagram/2021-09-17_13-47-58.png)

Verify the virtualservers created in the servicelist

![serverlist](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/externaldns/single-cluster/diagram/2021-09-17_13-50-05.png)

## Step 4: Create the WideIP's using the ExternalDNS CRDs

The diagram below show the **VirtualServer** and **ExternalDNS CRD** used in this user-guide. Important to **Note** the following:

* **Host:** **mysite.f5demo.com** in the **VirtualServer** CRD needs to match **domainName: mysite.f5demo.com** and **pools name: mysite.f5demo.com** **ExternalDNS CRD**
* Use the following string for the **GSLB monitor** in the ExternalDNS CRD

```
 monitor:
      type: http
      send: "GET / HTTP/1.1\r\nHost: mysite.f5demo.com\r\n"
```

* **dataServerName: /Common/big-ip-60-cluster** in the ExternalDNS CRD needs to match DataCenter Server name

![architecture](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/externaldns/single-cluster/diagram/2021-09-17_10-25-22.png)

Create the mysite and myapp EDNS CRDs

```
kubectl create -f edns-myapp.yaml
kubectl create -f edns-mysite.yaml
```

Validate the **WIDE IP list**. You should see both Wide IP created for mysite and myapp and both green status

![architecture](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/externaldns/single-cluster/diagram/2021-09-20_15-14-10.png)

If the status for the Wide IP's show red then maybe the **external DNS monitor** has failed. Check the monitor

![monitor](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/externaldns/single-cluster/diagram/2021-09-20_15-20-20.png)

Also validate the **Send String**

![sendstring](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/externaldns/single-cluster/diagram/2021-09-20_15-21-00.png)