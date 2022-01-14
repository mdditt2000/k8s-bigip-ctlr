# ExternalDNS for NGINX Ingress Controller using F5 CIS with BIG-IP DNS

The purpose of this document is to demonstrate ExternalDNS with NGINX Ingress Controller using F5 CIS with BIG-IP.  ExternalDNS allows user to control DNS records dynamically via Kubernetes CRD resources in a DNS provider-agnostic way. This user-guide documents ExternalDNS with F5 CIS + BIG-IP LTM and DNS, load balancing to NGINX Ingress Controller. BIG-IP LTM and DNS are configured on the same device for a single cluster as shown in the diagram. However BIG-IP LTM and DNS can be on dedicated devices for multiple sites,clusters and data centers. 

ExternalDNS solution provides you with modern, container application workloads that use both BIG-IP Container Ingress Services and NGINX Ingress Controller for Kubernetes. It’s an elegant control plane solution that offers a unified method of working with both technologies from a single interface—offering the best of BIG-IP and NGINX and fostering better collaboration across NetOps and DevOps teams. The diagram below demonstrates this use-case.

This architecture diagram demonstrates the ExternalDNS with NGINX Ingress Controller using F5 CIS with BIG-IP

![architecture](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/externaldns-nginx/diagram/2022-01-13_10-37-44.png)

Demo [YouTube](https://youtu.be/XdFqcCZWe5U)

This user-guide demonstrates a single Wide IP **cafe.example.com** which answers for both coffee and tea deployments. DNS has no layer 7 path awareness and therefore a DNS monitor is required to determine the health of the deployments. Recommendation would be to create a dedicated http status page for the DNS monitor, monitoring required deployments etc. IF the monitor detects the http status failure, the Wide IP is removed from BIG-IP DNS. Another option is to have a 1-1 mapping between the Wide IPand service. 

## Prerequisites

* Recommend AS3 version 3.30 [repo](https://github.com/F5Networks/f5-appsvcs-extension/releases/tag/v3.30.0)
* F5 CIS 2.7 [repo](https://github.com/F5Networks/k8s-bigip-ctlr/releases/tag/v2.7.0)
* Clouddocs [documentation](https://clouddocs.f5.com/containers/latest/userguide/crd/externaldns.html)

## Step 1: Deploy CIS

CIS 2.7 communicates directly with BIG-IP DNS via the Rest API and requires gtm-bigip-username and password. Since BIG-IP LTM and DNS are on the same device you can re-use the secret generic bigip-login when deploying CIS as shown below. In this example user-guide the Public IP address for BIGIP VirtualServer will be obtained from F5 IPAM. 

Add the following parameters to THE CIS deployment

* --custom-resource-mode=true - Configure CIS to only monitor CRDs. CIS will ignore all other resources
* --gtm-bigip-username - Provide username for CIS to access GTM
* --gtm-bigip-password - Provide password for CIS to access GTM
* --gtm-bigip-url - Provide url for CIS to access GTM. CIS uses the python SDK to configure GTM
* --ipam=true - CIS to pull VirtualServer IP address from IPAM range

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
    "--namespace=nginx-ingress",
    "--pool-member-type=cluster",
    "--flannel-name=fl-vxlan",
    "--insecure=true",
    "--custom-resource-mode=true",
    "--as3-validation=true",
    "--log-as3-response=true",
    "--ipam=true",
]
```

Deploy CIS in both locations

```
kubectl create secret generic bigip-login -n kube-system --from-literal=username=admin --from-literal=password=<secret>
kubectl create -f bigip-ctlr-clusterrole.yaml
kubectl create -f f5-bigip-ctlr-deployment.yaml
kubectl create -f f5-bigip-node.yaml
```
- f5-bigip-node is required for Flannel
- bigip-ctlr-clusterrole is required for CIS permissions 

cis-deployment [repo](https://github.com/mdditt2000/k8s-bigip-ctlr/tree/main/user_guides/externaldns-nginx/cis/cis-deployment)

## Step 2: F5 IPAM

CIS uses IPAM integration to provisions this external IP address on BIG-IP. F5 IPAM controller is optional for this solution. **--ipam=true** needs to be added to the CIS deployment and **ipamLabel: Test** and needs to be added to the VirtualServer. Following parameter are required for the IPAM deployment:

```
args:
  - --orchestration=kubernetes
  - --ip-range='{"Test":"10.192.75.117-10.192.75.119"}'
  - --log-level=DEBUG
```

Modify the persistent volume manifest file that meets your kubernetes deployment 

```
- ReadWriteOnce
  storageClassName: local-storage
  local:
    path: /tmp/cis_ipam
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - k8s-1-19-node1.example.com
```

### Deploy F5 IPAM Controller and Persistent Volumes deployment files

```
kubectl create -f f5-ipam-ctlr-rbac.yaml
kubectl create -f f5-ipam-persitentvolume.yaml
kubectl create -f f5-ipam-deployment.yaml
```
ipam-deployment [repo](https://github.com/mdditt2000/kubernetes-1-19/tree/master/cis%202.7.1/edns-multi-host/ipam-deployment)

View the following guide to help troubleshooting IPAM [IPAM](https://github.com/mdditt2000/kubernetes-1-19/tree/master/cis%202.7/ipam)

## Step 3: Nginx-Controller Installation

Deploy NGINX controller by using the following:

Create a namespace and a service account for the Ingress controller:
   
    kubectl apply -f nginx-config/ns-and-sa.yaml
   
Create a cluster role and cluster role binding for the service account:
   
    kubectl apply -f nginx-config/rbac.yaml
   
Create a secret with a TLS certificate and a key for the default server in NGINX:

    kubectl apply -f nginx-config/default-server-secret.yaml
    
Create a config map for customizing NGINX configuration:

    kubectl apply -f nginx-config/nginx-config.yaml
    
Create an IngressClass resource (for Kubernetes >= 1.18):  
    
    kubectl apply -f nginx-config/ingress-class.yaml
  
Create a service for the Ingress Controller pods for ports 80 and 443 as follows:

    kubectl apply -f nginx-config/nginx-service.yaml

nginx-config [repo](https://github.com/mdditt2000/kubernetes-1-19/tree/master/cis%202.7.1/edns-multi-host/nginx-config)

## Step 4: Deploy the Cafe Application

Create the coffee and the tea deployments and services:

    kubectl create -f cafe.yaml

#### Configure Load Balancing for the Cafe Application

Create a secret with an SSL certificate and a key:

    kubectl create -f cafe-secret.yaml

Create an Ingress resource:

    kubectl create -f cafe-ingress.yaml

ingress-example [repo](https://github.com/mdditt2000/kubernetes-1-19/tree/master/cis%202.7.1/edns-multi-host/ingress-example)

## Step 4: Create VirtualServer and ExternalDNS CRDs

Create the coffee and the tea VirtualServer CRD

    kubectl create -f vs-tea.yaml
    Kubectl create -f vs-coffee.yaml

Validated VirtualServer on BIG-IP

![VirtualServer](https://github.com/mdditt2000/kubernetes-1-19/blob/master/cis%202.7.1/edns-multi-host/diagram/2022-01-13_14-20-27.png)

Create the coffee and the tea ExternalDNS CRD

    kubectl create -f edns-tea.yaml
    Kubectl create -f edns-coffee.yaml

Validated VirtualServer CRD and ExternalCRD

**Note** IPAM has provided the external IP address for **cafe.example.com**. **hostGroup: "cafe"** is configured in the VirtualServer CRDs to maintain the same external IP address for all VirtualServer CRDs

```
❯ kubectl get crd,vs,externaldns -n nginx-ingress
NAME                                 HOST               TLSPROFILENAME   HTTPTRAFFIC   IPADDRESS   IPAMLABEL   IPAMVSADDRESS   STATUS   AGE
virtualserver.cis.f5.com/vs-coffee   cafe.example.com   reencrypt-tls    redirect                  Test        10.192.75.117   Ok       6d3h
virtualserver.cis.f5.com/vs-tea      cafe.example.com   reencrypt-tls    redirect                  Test        10.192.75.117   Ok       6d2h

NAME                                 DOMAINNAME         AGE     CREATED ON
externaldns.cis.f5.com/edns-coffee   cafe.example.com   2d15h   2022-01-11T06:36:44Z
externaldns.cis.f5.com/edns-tea      cafe.example.com   27h     2022-01-12T18:57:08Z                                               
```

Validated Wide IP on BIG-IP DNS

![Wide IP](https://github.com/mdditt2000/kubernetes-1-19/blob/master/cis%202.7.1/edns-multi-host/diagram/2022-01-13_14-29-16.png)