# F5 IngressLink with ExternalDNS

F5 IngressLink addresses modern app delivery at scale while resolving persona challenges. F5 IngressLink is a resource definition defined between BIG-IP and NGINX using F5 Container Ingress Service (CIS) and Nginx Ingress Controller (IC). F5 IngressLink with ExternalDNS now allows Application Developers to control DNS records dynamically via Kubernetes CRD resources in a DNS provider-agnostic way providing Active-Standby and Active-Active deployments. 

## Why F5 IngressLink

F5 IngressLink is integration between BIG-IP and NGINX technologies. F5 IngressLink was built to support customers with modern, container application workloads that use both BIG-IP, CIS and NGINX IC for Kubernetes. 

* IngressLink provides an **elegant control plane** solution that offers a unified method of working with F5 technologies
* Providing a single interface—offering the best of BIG-IP and NGINX and fostering better collaboration across **Infrastructure Providers**, **Cluster Operators** and **Application Developers** personas as shown below
* Aligning priorities and **minimize conflict** between teams while **enabling multi-tenancy** in Kubernetes
* Save time and **reduce errors** with declarative deployment and simplify integration between BIG-IP and NGINX

<img src="https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/ingresslink-externaldns/diagram/2022-03-10_14-50-09.png" width="428" height="441">

This user-guide documents IngressLink with ExternalDNS. BIG-IP LTM and DNS are configured on the same device for a single cluster as shown in the diagram. However BIG-IP LTM and DNS can be on dedicated devices for multiple sites,clusters and Data Centers. This architecture diagram demonstrates IngressLink with ExternalDNS

![architecture](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/ingresslink-externaldns/diagram/2022-03-10_16-44-07.png)

Demo on YouTube [video]()

On this page you’ll find:

* Links to the GitHub repositories for all the requisite software
* Documentation for the solution(s)
* A step by step configuration and deployment guide for F5 IngressLink with ExternalDNS

## IngressLink Compatibility Matrix

Recommended version for IngressLink:

* Recommend AS3 version 3.34 [repo](https://github.com/F5Networks/f5-appsvcs-extension/releases/tag/v3.34.0)
* CIS 2.8 [coming soon]()
* NGINX+ IC [repo](https://github.com/mdditt2000/k8s-bigip-ctlr/tree/main/user_guides/ingresslink/clusterip/nginx-config)
* Product Documentation [documentation](https://clouddocs.f5.com/containers/latest/userguide/ingresslink/)

## Configure F5 IngressLink with Kubernetes

**Step 1:**

### Create the  Proxy Protocol iRule on Bigip

Proxy Protocol is required by NGINX to provide the applications PODs with the original client IPs. Use the following steps to configure the Proxy_Protocol_iRule

* Login to BigIp GUI 
* On the Main tab, click Local Traffic > iRules.
* Click Create.
* In the Name field, type name as "Proxy_Protocol_iRule".
* In the Definition field, Copy the definition from "Proxy_Protocol_iRule" file. Click Finished.

proxy_protocol_iRule [repo](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/ingresslink-externaldns/big-ip/proxy-protocal/irule)

**Step 2**

### Install CIS

Add BIG-IP credentials as Kubernetes Secrets

    kubectl create secret generic bigip-login -n kube-system --from-literal=username=admin --from-literal=password=<password>

Create a Service Account, Cluster Role and Cluster Role Binding

    kubectl create -f bigip-ctlr-clusterrole.yaml
    
Create CRD schema

    kubectl create -f customresourcedefinition.yaml

cis-crd-schema [repo](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/ingresslink-externaldns/cis/cis-crd-schema/customresourcedefinitions.yml)

Update the bigip address, partition and other details(image, imagePullSecrets, etc) in CIS deployment file and Install CIS Controller in ClusterIP mode as follows:

* Add the following statements to the CIS deployment arguments for Ingresslink with ExternalDNS

```
args: [
  # See the k8s-bigip-ctlr documentation for information about
  # all config options
  # https://clouddocs.f5.com/containers/latest/
    "--bigip-username=$(BIGIP_USERNAME)",
    "--bigip-password=$(BIGIP_PASSWORD)",
    "--bigip-url=192.168.200.60",
    "--bigip-partition=ingresslink",
    "--namespace=nginx-ingress",
    "--gtm-bigip-username=$(BIGIP_USERNAME)",
    "--gtm-bigip-password=$(BIGIP_PASSWORD)",
    "--gtm-bigip-url=192.168.200.60",
    "--pool-member-type=cluster",
    "--flannel-name=fl-vxlan",
    "--log-level=INFO",
    "--insecure=true",
    "--custom-resource-mode=true",
]
```

* To deploy the CIS controller in cluster mode update CIS deployment arguments as follows for kubernetes.

    - "--pool-member-type=cluster"
    - "--flannel-name=fl-vxlan"

Additionally, if you are deploying the CIS in Cluster Mode you need to have following prerequisites. For more information, see [Deployment Options](https://clouddocs.f5.com/containers/latest/userguide/config-options.html#config-options)
    
* You must have a fully active/licensed BIG-IP. SDN must be licensed. For more information, see [BIG-IP VE license support for SDN services](https://support.f5.com/csp/article/K26501111).
* VXLAN tunnel should be configured from Kubernetes Cluster to BIG-IP. For more information see, [Creating VXLAN Tunnels](https://clouddocs.f5.com/containers/latest/userguide/cis-helm.html#creating-vxlan-tunnels)

#### Create CIS using the below manifest

```
kubectl create -f f5-bigip-ctlr-deployment.yaml
```

cis-deployment [repo](https://github.com/mdditt2000/k8s-bigip-ctlr/tree/main/user_guides/ingresslink-externaldns/cis/cis-deployment)

#### Verify CIS deployment

```
❯ kubectl get deploy/k8s-bigip-ctlr-deployment -n kube-system
NAME                        READY   UP-TO-DATE   AVAILABLE   AGE
k8s-bigip-ctlr-deployment   1/1     1            1           3h51m
```

**Step 3**

### Nginx-Controller Installation

Create NGINX IC custom resource definitions for VirtualServer and VirtualServerRoute, TransportServer and Policy resources:

    kubectl apply -f k8s.nginx.org_virtualservers.yaml
    kubectl apply -f k8s.nginx.org_virtualserverroutes.yaml
    kubectl apply -f k8s.nginx.org_transportservers.yaml
    kubectl apply -f k8s.nginx.org_policies.yaml

crd-schema [repo](https://github.com/nginxinc/kubernetes-ingress/tree/v1.10.0/deployments/common/crds)

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

Use a Deployment. When you run the Ingress Controller by using a Deployment, by default, Kubernetes will create one Ingress controller pod.
    
    kubectl apply -f nginx-config/nginx-ingress.yaml
  
Create a service for the Ingress Controller pods for ports 80 and 443 as follows:

    kubectl apply -f nginx-config/nginx-service.yaml

Verify NGINX-Ingress deployment

```
❯ kubectl get deploy/nginx-ingress -n nginx-ingress
NAME            READY   UP-TO-DATE   AVAILABLE   AGE
nginx-ingress   4/4     4            4           5h33m
```
nginx-config [repo](https://github.com/mdditt2000/k8s-bigip-ctlr/tree/main/user_guides/ingresslink-externaldns/nginx-config)

## Deploy the Cafe Application

**Step 4**

Create the coffee and the tea deployments and services:

    kubectl create -f cafe.yaml

### Configure Load Balancing for the Cafe Application

Create a secret with an SSL certificate and a key:

    kubectl create -f cafe-secret.yaml

Create an Ingress resource:

    kubectl create -f cafe-ingress.yaml

demo application [repo](https://github.com/mdditt2000/k8s-bigip-ctlr/tree/main/user_guides/ingresslink-externaldns/ingress-example)

## Create an IngressLink and ExternalDNS CRD

**Step 5**

Add the **Public-IP** and **Host** to the IngressLink CRD. Host is **Wide-IP** that will match the ExternalDNS crd as shown below in the diagram

![crd]https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/ingresslink-externaldns/diagram/2022-03-10_17-10-00.png)

#### Create the IngressLink CRD

 ```
❯ kubectl create -f vs-ingresslink.yaml
ingresslink.cis.f5.com/vs-ingresslink created
```

**Note:** The name of the app label selector in IngressLink CRD should match the labels of the **nginx-ingress service** created in step-3. Recommend keeping the deployment default.

#### Validate the Public IP address on BIG-IP

![crd](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/ingresslink-externaldns/diagram/2022-03-10_17-20-43.png)

#### Validate the BIG-IP Pools

![crd](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/ingresslink-externaldns/diagram/2022-03-10_17-21-36.png)

#### Validate the Proxy-Protocol iRule

![crd](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/ingresslink-externaldns/diagram/2022-03-10_17-22-03.png)

#### Create the ExternalDNS CRD

 ```
❯ kubectl create -f edns-cafe.yaml
externaldns.cis.f5.com/edns-cafe created
```

crd-resource [repo](https://github.com/mdditt2000/k8s-bigip-ctlr/tree/main/user_guides/ingresslink-externaldns/cis/crd-resource)

#### Validate the ExternalDNS CRD in Kubernetes

```
❯ kubectl get crd,ingresslink,externaldns -n nginx-ingress
NAME                                    IPAMVSADDRESS   AGE
ingresslink.cis.f5.com/nginx-ingress                    4h25m
ingresslink.cis.f5.com/vs-ingresslink                   4h19m

NAME                               DOMAINNAME         AGE     CREATED ON
externaldns.cis.f5.com/edns-cafe   cafe.example.com   4h12m   2022-03-10T21:17:36Z
```

#### Validate the Wide-IP on BIG0-IP DNS

![crd](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/ingresslink-externaldns/diagram/2022-03-10_17-31-50.png)

#### Validate the Wide-IP Pool on BIG0-IP DNS

![crd](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/ingresslink-externaldns/diagram/2022-03-10_17-40-00.png)

**Step 6**

### Test the Application

Connect to the host via the browser

![traffic](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/ingresslink-externaldns/diagram/2022-03-10_17-43-16.png)