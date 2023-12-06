# Multi-Cluster NGINX Gateway API Fabric with F5 BIG-IP

This repo documents how to reduce complexity for your Kubernetes apps with the **Gateway API-conformant NGINX Gateway Fabric**. Also using F5 BIG-IP and Container Ingress Services (CIS) to as the **public entry point to the multiple Kubernetes clusters** as shown in the diagram below:

Demo on YouTube [video]()

![diagram](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/multi-cluster-nginx-gateway-fabric/diagram/2023-12-06_15-03-42.png)

The link walks you through how to install NGINX Gateway Fabric on a Kubernetes multi-cluster with some changes

* Install NGINX Gateway Fabric [docs](https://docs.nginx.com/nginx-gateway-fabric/installation/)

## Expose NGINX Gateway Fabric

You can gain access to NGINX Gateway Fabric by creating a Service with F5 CIS. This Service must live in the same Namespace **nginx-gateway** as the controller. CIS will configure BIG-IP with a Public IP address and load balance all traffic to the NGINX Gateway Fabric in the multiple Kubernetes clusters. BIG-IP routes traffic to the NGINX Gateway Fabric using the Calico CNI. Therefore BIG-IP can use ClusterIP instead of NodePort. This is by far a superior Kubernetes implementation. Change the type to service to ClusterIP as shown in the example

```
---
# Source: nginx-gateway-fabric/templates/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-gateway
  namespace: nginx-gateway
  labels:
    app.kubernetes.io/name: nginx-gateway
    app.kubernetes.io/instance: nginx-gateway
    app.kubernetes.io/version: "edge"
spec:
  type: ClusterIP  ---------------------------- Change this value
  selector:
    app.kubernetes.io/name: nginx-gateway
    app.kubernetes.io/instance: nginx-gateway
  ports: # Update the following ports to match your Gateway Listener ports
  - name: http
    port: 80
    protocol: TCP
    targetPort: 80
  - name: https
    port: 443
    protocol: TCP
    targetPort: 443
```

No need to setup type of **loadbalancer** unless you want CIS to populate the Public IP. In this example CIS configures BIG-IP using a VirtualServer CRD as shown in the example below:

```
apiVersion: "cis.f5.com/v1"
kind: VirtualServer
metadata:
  name: nginx-gateway-80
  namespace: nginx-gateway
  labels:
    f5cr: "true"
spec:
  virtualServerAddress: 10.192.75.121
  host: cafe.example.com
  pools:
  - path: /
    service: nginx-gateway
    servicePort: 80
    extendedServiceReferences:
        - clusterName: kubeconfig-1-28
          namespace: nginx-gateway
          port: 80
          serviceName: nginx-gateway
```
Since NGINX Gateway Fabric is set to 2 replicas in both blusters, BIG-IP will see 2 pods per cluster. Total of 4 pods in the BIG-IP pool as shown below. 

![pool](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/multi-cluster-nginx-gateway-fabric/diagram/2023-12-06_14-44-45.png)





