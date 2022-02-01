# Per Application Failover using ExternalDNS

The purpose of this document is to demonstrate per application failover using ExternalDNS with NGINX Ingress Controller and F5 BIG-IP. ExternalDNS allows user to control DNS records dynamically via Kubernetes CRD resources in a DNS provider-agnostic way. This user-guide documents ExternalDNS with F5 CIS + BIG-IP LTM and DNS, load balancing to NGINX Ingress Controller. BIG-IP LTM and DNS are configured on the same device for a single cluster as shown in the diagram. However BIG-IP LTM and DNS can be on dedicated devices for multiple sites,clusters and data centers. 

ExternalDNS solution provides you with modern, container application workloads that use both F5 Container Ingress Services and NGINX Ingress Controller for Kubernetes. It’s an elegant control plane solution that offers a unified method of working with both technologies from a single interface—offering the best of BIG-IP and NGINX and fostering better collaboration across NetOps and DevOps teams. The diagram below demonstrates this use-case.

This architecture diagram demonstrates the ExternalDNS with NGINX Ingress Controller using F5 CIS with BIG-IP for ten applications

![architecture](https://github.com/mdditt2000/kubernetes-1-19/blob/master/cis%202.7.1/per-application-failover/diagram/2022-01-31_14-03-56.png)

Demo [YouTube]()

This user-guide demonstrates ten applications, each application having a unique Public Wide IP's **HOST name** which answers for the deployment in Kubernetes. DNS has no layer 7 path awareness and therefore a DNS monitor is required to determine the health of the deployments. Each ExternalDNS CRD would specify the DNS monitor on BIG-IP. Recommended to work with your F5 Solution Architect to discuss DNS monitor scaling. If the monitor detects the http status failure, the Wide IP is removed from BIG-IP DNS.

## Prerequisites

* Recommend AS3 version 3.30 or later [repo](https://github.com/F5Networks/f5-appsvcs-extension/releases/tag/v3.30.0)
* F5 CIS 2.7 [repo](https://github.com/F5Networks/k8s-bigip-ctlr/releases/tag/v2.7.0)
* Clouddocs [documentation](https://clouddocs.f5.com/containers/latest/userguide/crd/externaldns.html)

## Step 1: Deploy CIS

CIS 2.7 communicates directly with BIG-IP DNS via the Rest API and requires gtm-bigip-username and password. Since BIG-IP LTM and DNS are on the same device you can re-use the secret generic bigip-login when deploying CIS as shown below. In this example user-guide the Public IP addresses will be specified by the VirtualServer CRD. 

Add the following parameters to THE CIS deployment

* --custom-resource-mode=true - Configure CIS to only monitor CRDs. CIS will ignore all other resources
* --gtm-bigip-username - Provide username for CIS to access GTM
* --gtm-bigip-password - Provide password for CIS to access GTM
* --gtm-bigip-url - Provide url for CIS to access GTM. CIS uses the python SDK to configure GTM

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
]
```

Deploy CIS

```
kubectl create secret generic bigip-login -n kube-system --from-literal=username=admin --from-literal=password=<secret>
kubectl create -f bigip-ctlr-clusterrole.yaml
kubectl create -f f5-bigip-ctlr-deployment.yaml
kubectl create -f f5-bigip-node.yaml
```
- f5-bigip-node is required for Flannel
- bigip-ctlr-clusterrole is required for CIS permissions 

cis-deployment [repo](https://github.com/mdditt2000/kubernetes-1-19/tree/master/cis%202.7.1/per-application-failover/cis/cis-deployment)

## Step 2: Nginx-Controller Installation

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

## Step 3: Deploy the Cafe Applications

Create the ten **cafe** deployments and services:

```
kubectl create -f cafe.yaml
```

### Configure Load Balancing for the Cafe Application

Create a secret with an SSL certificate and a key:

```
kubectl create -f cafe-secret.yaml
```

Create an ten Ingress resource for the **cafe** applications:

```
kubectl create -f brew-ingress.yaml
kubectl create -f chai-ingress.yaml
kubectl create -f coffee-ingress.yaml
kubectl create -f flatwhite-ingress.yaml
kubectl create -f frappuccino-ingress.yaml
kubectl create -f macchiato-ingress.yaml
kubectl create -f mocha-ingress.yaml
kubectl create -f smoothie-ingress.yaml
kubectl create -f tea-ingress.yaml
```

View the Ingress resources for the **cafe** applications:

```
❯ kubectl get ingress
NAME                  CLASS    HOSTS                     ADDRESS   PORTS     AGE
brew-ingress          <none>   brew.example.com                    80, 443   21h
chai-ingress          <none>   chai.example.com                    80, 443   21h
coffee-ingress        <none>   coffee.example.com                  80, 443   3d20h
flatwhite-ingress     <none>   flatwhite.example.com               80, 443   21h
frappuccino-ingress   <none>   frappuccino.example.com             80, 443   21h
latte-ingress         <none>   latte.example.com                   80, 443   3d18h
macchiato-ingress     <none>   macchiato.example.com               80, 443   21h
mocha-ingress         <none>   mocha.example.com                   80, 443   3d18h
smoothie-ingress      <none>   smoothie.example.com                80, 443   21h
tea-ingress           <none>   tea.example.com                     80, 443   3d20h
```

ingress-example [repo](https://github.com/mdditt2000/kubernetes-1-19/tree/master/cis%202.7.1/per-application-failover/ingress-example)

## Step 4: Create TLSProfile CRDs for re-encryptions

The diagram below demonstrates the combination mapping of the VirtualServer, TLSProfile, ExternalDNS CRDs for application **tea.example.com**:

- Reencrypt Termination
- Redirect HTTP to HTTPS
- SNI HTTPS Health monitor of the backend application using HOST **tea.example.com** and **PATH /tea**
- End-to-end-TLS
- Re-use the same **virtualServerAddress: "10.192.75.117"** for all cafe application
- Hostname load balancing based on the HOST **tea.example.com** and **PATH /tea**

![combo](https://github.com/mdditt2000/kubernetes-1-19/blob/master/cis%202.7.1/per-application-failover/diagram/2022-02-01_12-31-07.png)

Create the **cafe** TLSProfile CRDs

```
kubectl create -f reencrypt-brew.yaml
Kubectl create -f reencrypt-chai.yaml
kubectl create -f reencrypt-coffee.yaml
Kubectl create -f reencrypt-flatwhite.yaml
kubectl create -f reencrypt-frappuccino.yaml
Kubectl create -f reencrypt-latte.yaml
kubectl create -f reencrypt-macchiato.yaml
Kubectl create -f reencrypt-mocha.yaml
kubectl create -f reencrypt-smoothie.yaml
Kubectl create -f reencrypt-tea.yaml.yaml
```

TLSProfile [repo](https://github.com/mdditt2000/kubernetes-1-19/tree/master/cis%202.7.1/per-application-failover/cis/cafe/reencrypt)

Create the **cafe** VirtualServer CRDs

```
kubectl create -f vs-brew.yaml
Kubectl create -f vs-chai.yaml
kubectl create -f vs-coffee.yaml
Kubectl create -f vs-flatwhite.yaml
kubectl create -f vs-frappuccino.yaml
Kubectl create -f vs-latte.yaml
kubectl create -f vs-macchiato.yaml
Kubectl create -f vs-mocha.yaml
kubectl create -f vs-smoothie.yaml
Kubectl create -f vs-tea.yaml.yaml
```

VirtualServer [repo](https://github.com/mdditt2000/kubernetes-1-19/tree/master/cis%202.7.1/per-application-failover/cis/cafe/virtualserver)

Validated the **VirtualServer** and **TLSProfile** CRDs in Kubernetes

```
❯ kubectl get vs,TLSProfile -n nginx-ingress
NAME                                      HOST                      TLSPROFILENAME          HTTPTRAFFIC   IPADDRESS       IPAMLABEL   IPAMVSADDRESS   STATUS   AGE
virtualserver.cis.f5.com/vs-brew          brew.example.com          reencrypt-brew          redirect      10.192.75.117               None            Ok       20h
virtualserver.cis.f5.com/vs-chai          chai.example.com          reencrypt-chai          redirect      10.192.75.117               None            Ok       20h
virtualserver.cis.f5.com/vs-coffee        coffee.example.com        reencrypt-coffee        redirect      10.192.75.117               None            Ok       20h
virtualserver.cis.f5.com/vs-flatwhite     flatwhite.example.com     reencrypt-flatwhite     redirect      10.192.75.117               None            Ok       20h
virtualserver.cis.f5.com/vs-frappuccino   frappuccino.example.com   reencrypt-frappuccino   redirect      10.192.75.117               None            Ok       20h
virtualserver.cis.f5.com/vs-latte         latte.example.com         reencrypt-latte         redirect      10.192.75.117               None            Ok       20h
virtualserver.cis.f5.com/vs-macchiato     macchiato.example.com     reencrypt-macchiato     redirect      10.192.75.117               None            Ok       20h
virtualserver.cis.f5.com/vs-mocha         mocha.example.com         reencrypt-mocha         redirect      10.192.75.117               None            Ok       20h
virtualserver.cis.f5.com/vs-smoothie      smoothie.example.com      reencrypt-smoothie      redirect      10.192.75.117               None            Ok       19h
virtualserver.cis.f5.com/vs-tea           tea.example.com           reencrypt-tea           redirect      10.192.75.117               None            Ok       19h

NAME                                          AGE
tlsprofile.cis.f5.com/reencrypt-brew          20h
tlsprofile.cis.f5.com/reencrypt-chai          20h
tlsprofile.cis.f5.com/reencrypt-coffee        20h
tlsprofile.cis.f5.com/reencrypt-flatwhite     20h
tlsprofile.cis.f5.com/reencrypt-frappuccino   20h
tlsprofile.cis.f5.com/reencrypt-latte         20h
tlsprofile.cis.f5.com/reencrypt-macchiato     20h
tlsprofile.cis.f5.com/reencrypt-mocha         20h
tlsprofile.cis.f5.com/reencrypt-smoothie      20h
tlsprofile.cis.f5.com/reencrypt-tea           20h
```
**Note** Make sure the status is OK. This is a verification that the LTM profile is created on BIG-IP

Review LTM load balancing profiles on BIG-IP LTM

![BIG-IP LTM](https://github.com/mdditt2000/kubernetes-1-19/blob/master/cis%202.7.1/per-application-failover/diagram/2022-02-01_11-42-23.png)

Create the **cafe** ExternalDNS CRDs

```
kubectl create -f vs-brew.yaml
Kubectl create -f vs-chai.yaml
kubectl create -f vs-coffee.yaml
Kubectl create -f vs-flatwhite.yaml
kubectl create -f vs-frappuccino.yaml
Kubectl create -f vs-latte.yaml
kubectl create -f vs-macchiato.yaml
Kubectl create -f vs-mocha.yaml
kubectl create -f vs-smoothie.yaml
Kubectl create -f vs-tea.yaml.yaml
```

ExternalDNS [repo](https://github.com/mdditt2000/kubernetes-1-19/tree/master/cis%202.7.1/per-application-failover/cis/cafe/externaldns) 

Validated ExternalDNS CRDs

```
kubectl get externaldns -n nginx-ingress
NAME               DOMAINNAME                AGE   CREATED ON
edns-brew          brew.example.com          20h   2022-01-31T23:40:18Z
edns-chai          chai.example.com          20h   2022-01-31T23:17:07Z
edns-coffee        coffee.example.com        20h   2022-01-31T23:09:28Z
edns-flatwhite     flatwhite.example.com     20h   2022-01-31T23:18:46Z
edns-frappuccino   frappuccino.example.com   20h   2022-01-31T23:46:15Z
edns-latte         latte.example.com         20h   2022-01-31T23:21:09Z
edns-macchiato     macchiato.example.com     19h   2022-01-31T23:47:25Z
edns-mocha         mocha.example.com         20h   2022-01-31T23:43:44Z
edns-smoothie      smoothie.example.com      19h   2022-01-31T23:49:56Z
edns-tea           tea.example.com           19h   2022-01-31T23:50:45Z                                      
```

Validated Wide IP on BIG-IP DNS

![Wide IP](https://github.com/mdditt2000/kubernetes-1-19/blob/master/cis%202.7.1/per-application-failover/diagram/2022-01-31_15-55-49.png)

Validated DNS monitor for application **tea.example.com**

![Wide IP](https://github.com/mdditt2000/kubernetes-1-19/blob/master/cis%202.7.1/per-application-failover/diagram/2022-02-01_12-50-26.png)

## Step 5: Connect to Multiple Applications

In this example we connecting to multiple applications based on their FQDN. The BIG-IP DNS is designated **nameserver** for the **A recorded**

![DNS](https://github.com/mdditt2000/kubernetes-1-19/blob/master/cis%202.7.1/per-application-failover/diagram/2022-02-01_12-55-20.png)

Connect to the Public IP via the FQDN and application Path. Example below of /tea, /coffee and /mocha

![traffic](https://github.com/mdditt2000/kubernetes-1-19/blob/master/cis%202.7.1/per-application-failover/diagram/2022-02-01_13-02-20.png)

## Step 6: Fail Application POD in Kubernetes

To demonstrate an application failure we going to replicate the application pods to zero. WIDE IPs on the BIG-IP DNS will be removed

```
kubectl scale deploy/tea --replicas=0
kubectl scale deploy/coffee --replicas=0
kubectl scale deploy/mocha --replicas=0
```
![failure](https://github.com/mdditt2000/kubernetes-1-19/blob/master/cis%202.7.1/per-application-failover/diagram/2022-02-01_13-13-04.png)

Demonstrate the failed monitor

![monitor](https://github.com/mdditt2000/kubernetes-1-19/blob/master/cis%202.7.1/per-application-failover/diagram/2022-02-01_13-17-24.png)