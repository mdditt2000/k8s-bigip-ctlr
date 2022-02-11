# Kube-Vip and F5 BIG-IP with NGINX Ingress Controller

The purpose of this document is to demonstrate Kube-VIP using BGP with NGINX Ingress Controller and F5 BIG-IP. Kube-vip provides Kubernetes clusters with a virtual IP and load balancer for both the control plane (for building a highly-available cluster) and Kubernetes Services of type LoadBalancer using external BIG-IP. This document focuses mainly on the Kubernetes Services of type LoadBalancer. The original purpose of kube-vip was to simplify the building of highly-available (HA) Kubernetes clusters, which at the time involved a few components and configurations that all needed to be managed. Since the project has evolved, it can now use those same technologies to provide load balancing capabilities for Kubernetes Service resources of type LoadBalancer. 

Using type LoadBalancer with NGINX Ingress Controller this allow a virtual IP to load balancer all traffic across the NGINX cluster to the applications pods. This virtual IP can be advertised by BIG-IP using BPG or other routing protocols. This architecture diagram demonstrates the kube-VIP using BGP with NGINX Ingress Controller and F5 BIG-IP

![architecture](https://github.com/mdditt2000/kubernetes-1-19/blob/master/cis%202.7.1/kube-vip/diagram/2022-02-10_11-13-59.png)

Demo [YouTube]()

This user-guide demonstrates a virtual IP which is updated by the kube-vip Cloud Provider. Kube-vip Cloud Provider monitors the Nginx-Ingress Service which is of type LoadBalancer and update the EXTERNAL-IP with the virtual IP. This virtual IP can be advertised to the BIG-IP BPG and redistributed to the network. 

## Step 1: Enable BGP on BIG-IP

Add BGP to route-domain

![bgp](https://github.com/mdditt2000/kubernetes-1-19/blob/master/cis%202.7.1/kube-vip/diagram/2022-02-10_13-56-44.png)

Access the IMI Shell

    imish

Switch to enable mode

    enable

Enter configuration mode

    config terminal

Setup route bgp with AS Number 65000

    router bgp 65000

Create BGP Peer group

    neighbor kube-vip-k8s peer-group

Assign peer group as BGP neighbors

    neighbor kube-vip-k8s remote-as 65000

Add all the peers: the other BIG-IP, our k8s components

    neighbor 192.168.200.61 peer-group kube-vip-k8s

Save configuration

    write

## Step 2: Deploy Kube-VIP

In this user-guide, I deployed Kube-VIP as a DaemonSet. The installation documentation can be located **daemonset** [repo](https://kube-vip.chipzoller.dev/docs/installation/daemonset/)

```
- name: bgp_peeras
    value: "65000"
- name: bgp_peers
    value: "192.168.200.60:65000::false"
- name: address
    value: 192.168.200.117
```

Deploy Kube-vip daemonset

```
kubectl apply -f https://kube-vip.io/manifests/rbac.yaml
kubectl create -f kube-vip-ds.yaml
``` 

kube-vip-daemonset [repo](https://github.com/mdditt2000/kubernetes-1-19/tree/master/cis%202.7.1/edns-multi-host/cis/cis-deployment)

```
❯ kubectl logs -f kube-vip-ds-tjttm -n kube-system
time="2022-02-10T21:29:59Z" level=info msg="server started"
time="2022-02-10T21:29:59Z" level=info msg="Starting Kube-vip Manager with the BGP engine"
time="2022-02-10T21:29:59Z" level=info msg="Namespace [kube-system], Hybrid mode [true]"
time="2022-02-10T21:29:59Z" level=info msg="Starting the BGP server to advertise VIP routes to BGP peers"
time="2022-02-10T21:29:59Z" level=info msg="Add a peer configuration for:192.168.200.60" Topic=Peer
time="2022-02-10T21:29:59Z" level=info msg="Beginning watching services for type: LoadBalancer in all namespaces"
time="2022-02-10T21:29:59Z" level=info msg="Service [kubernetes] has been added/modified, it has no assigned external addresses"
time="2022-02-10T21:29:59Z" level=info msg="Service [nginx-ingress] has been added/modified, it has an assigned external addresses [192.168.200.14]"
time="2022-02-10T21:29:59Z" level=info msg="New VIP [192.168.200.14] for [nginx-ingress/b15b1a7f-f3c0-4a94-a154-2906f936578b] "
time="2022-02-10T21:29:59Z" level=info msg="Starting advertising address [192.168.200.14] with kube-vip"
time="2022-02-10T21:29:59Z" level=info msg="Started Load Balancer and Virtual IP"
time="2022-02-10T21:29:59Z" level=info msg="Service [coffee-svc] has been added/modified, it has no assigned external addresses"
time="2022-02-10T21:29:59Z" level=info msg="Service [tea-svc] has been added/modified, it has no assigned external addresses"
time="2022-02-10T21:29:59Z" level=info msg="Service [kube-dns] has been added/modified, it has no assigned external addresses"
time="2022-02-10T21:30:07Z" level=info msg="Peer Up" Key=192.168.200.60 State=BGP_FSM_OPENCONFIRM Topic=Peer
2022/02/10 21:30:07 conf:<local_as:65000 neighbor_address:"192.168.200.60" peer_as:65000 > state:<local_as:65000 neighbor_address:"192.168.200.60" peer_as:65000 session_state:ESTABLISHED router_id:"192.168.200.60" > transport:<local_address:"192.168.200.61" local_port:36952 remote_port:179 >
```

## Step 3: Deploy kube-vip Cloud Provider

kube-vip is designed to be as decoupled or agnostic from other components that may exist within a Kubernetes cluster as possible. The kube-vip cloud provider can be used to populate an IP address for Services of type LoadBalancer similar to what public cloud providers allow through a Kubernetes CCM.

Install the kube-vip Cloud Provider

```
kubectl create -f kube-vip-cloud-controller.yaml
```

## Step 4: Deploy Nginx-Controller Installation

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

**NOTE** expose the nginx-ingress Service and once the cloud provider has provided an address, kube-vip will start to advertise that address to the outside world 

```
apiVersion: v1
kind: Service
metadata:
  name: nginx-ingress
  namespace: nginx-ingress
  labels:
    app: nginx-ingress
spec:
  ports:
    - port: 80
      targetPort: 80
      protocol: TCP
      name: http
    - port: 443
      targetPort: 443
      protocol: TCP
      name: https
  selector:
    app: nginx-ingress
  type: LoadBalancer
```

nginx-config [repo](https://github.com/mdditt2000/kubernetes-1-19/tree/master/cis%202.7.1/kube-vip/nginx-config)

## Step 5: Deploy the Cafe Application

Create the coffee and the tea deployments and services:

    kubectl create -f cafe.yaml

#### Configure Load Balancing for the Cafe Application

Create a secret with an SSL certificate and a key:

    kubectl create -f cafe-secret.yaml

Create an Ingress resource:

    kubectl create -f cafe-ingress.yaml

ingress-example [repo](https://github.com/mdditt2000/kubernetes-1-19/tree/master/cis%202.7.1/kube-vip/ingress-example)

## Step 6: Expose a Service

We can now expose a Service and once the cloud provider has provided an address, kube-vip will start to advertise that address in BGP. My example is very simple using a Static host and therefore i cannot take advantage of ConfigMap. Rather the virtual IP is comes from the interface.

Validate Kube-vip pods are deployed

```
❯ kubectl get pod -n kube-system | grep kube-vip
kube-vip-cloud-provider-0                                  1/1     Running   0          21h
kube-vip-ds-kjg7z                                          1/1     Running   0          21h
```

Log Kube-vip pod. You will notice the following:

- Kube-vip Manager with the BGP engine
- Advertise VIP routes to BGP peers **BIP-IP**
- Beginning watching services for type: LoadBalancer in all namespaces
- [nginx-ingress] has been added/modified, it has an assigned external addresses [192.168.200.14]
- Starting advertising address [192.168.200.14] with kube-vip"

```
time="2022-02-10T23:57:13Z" level=info msg="server started"
time="2022-02-10T23:57:13Z" level=info msg="Starting Kube-vip Manager with the BGP engine"
time="2022-02-10T23:57:13Z" level=info msg="Namespace [kube-system], Hybrid mode [false]"
time="2022-02-10T23:57:13Z" level=info msg="Starting the BGP server to advertise VIP routes to BGP peers"
time="2022-02-10T23:57:13Z" level=info msg="Add a peer configuration for:192.168.200.60" Topic=Peer
time="2022-02-10T23:57:13Z" level=info msg="Beginning watching services for type: LoadBalancer in all namespaces"
time="2022-02-10T23:57:13Z" level=info msg="Service [coffee-svc] has been added/modified, it has no assigned external addresses"
time="2022-02-10T23:57:13Z" level=info msg="Service [tea-svc] has been added/modified, it has no assigned external addresses"
time="2022-02-10T23:57:13Z" level=info msg="Service [kube-dns] has been added/modified, it has no assigned external addresses"
time="2022-02-10T23:57:13Z" level=info msg="Service [kubernetes] has been added/modified, it has no assigned external addresses"
time="2022-02-10T23:57:13Z" level=info msg="Service [nginx-ingress] has been added/modified, it has an assigned external addresses [192.168.200.14]"
time="2022-02-10T23:57:13Z" level=info msg="New VIP [192.168.200.14] for [nginx-ingress/b15b1a7f-f3c0-4a94-a154-2906f936578b] "
time="2022-02-10T23:57:13Z" level=info msg="Starting advertising address [192.168.200.14] with kube-vip"
time="2022-02-10T23:57:13Z" level=info msg="Started Load Balancer and Virtual IP"
time="2022-02-10T23:57:19Z" level=info msg="Peer Up" Key=192.168.200.60 State=BGP_FSM_OPENCONFIRM Topic=Peer
2022/02/10 23:57:19 conf:<local_as:65000 neighbor_address:"192.168.200.60" peer_as:65000 > state:<local_as:65000 neighbor_address:"192.168.200.60" peer_as:65000 session_state:ESTABLISHED router_id:"192.168.200.60" > transport:<local_address:"192.168.200.61" local_port:47466 remote_port:179 >
```

Validate the **nginx-service**

```
❯ kubectl get service -n nginx-ingress
NAME            TYPE           CLUSTER-IP       EXTERNAL-IP      PORT(S)                      AGE
nginx-ingress   LoadBalancer   10.103.107.192   192.168.200.14   80:30249/TCP,443:30029/TCP   2d22h
```

Validate BIG-IP routing table

```
east.f5demo.com[0]#show ip route
Codes: K - kernel, C - connected, S - static, R - RIP, B - BGP
       O - OSPF, IA - OSPF inter area
       N1 - OSPF NSSA external type 1, N2 - OSPF NSSA external type 2
       E1 - OSPF external type 1, E2 - OSPF external type 2
       i - IS-IS, L1 - IS-IS level-1, L2 - IS-IS level-2, ia - IS-IS inter area
       * - candidate default

Gateway of last resort is 10.192.75.1 to network 0.0.0.0

K*      0.0.0.0/0 [0/0] via 10.192.75.1, external
C       10.192.75.0/24 is directly connected, external
C       10.244.0.0/16 is directly connected, fl-vxlan
C       127.0.0.1/32 is directly connected, lo
C       127.1.1.254/32 is directly connected, tmm
C       192.168.200.0/24 is directly connected, internal
B       192.168.200.14/32 [200/0] via 192.168.200.61, internal, 01:11:11
B       192.168.200.117/32 [200/0] via 192.168.200.61, internal, 01:11:11
east.f5demo.com[0]#
Vty connection is timed out.
```
 
## Step 7: Connect to the Virtual IP

![traffic](https://github.com/mdditt2000/kubernetes-1-19/blob/master/cis%202.7.1/kube-vip/diagram/2022-02-11_14-03-10.png)


