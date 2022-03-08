# Multi-Cluster OpenShift/Kubernetes using BIG-IP DNS Failover with Nginx Ingress Controller

Today, organizations are increasingly deploying multiple container environment. Deploying multiple environment can improve availability, isolation and scalability. This user-guide demonstrates how F5 Container Ingress Services (CIS) can automate BIP-IP with Nginx Ingress Controller, to provide Ingress services for multiple container environments using DNS failover

## OpenShift and Kubernetes Container Environments

In this user-guide, we have deployed an OpenShift and Kubernetes container environments running identical applications front-ended by NGINX Ingress Controller. BIG-IP is platform-agnostic, using DNS to distribute traffic between the OpenShift and Kubernetes clusters. This simple but powerful approach enables users the flexibility to complete an container environment proof of concept or migrating applications between environments. Since CIS uses the Kubernetes API the resource definitions for OpenShift and Kubernetes are identical except for the public IPs. Diagram below represents the OpenShift and Kubernetes environments.

![architecture](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/multi-deployment-nginx/diagram/2022-02-28_11-11-06.png)

Demo on YouTube [video]()

This user-guide demonstrates an application having a Wide IP's HOST name **cafe.example.com**  which answers using **round-robin** for the OpenShift or Kubernetes environments. DNS has no layer seven path awareness and therefore DNS monitors are required to determine the health of the applications **/coffee** and **/tea**. Each ExternalDNS CRD specifies the DNS monitors on BIG-IP. Recommended to work with your F5 Solution Architect to discuss DNS monitoring and scaling. If a monitor detects the http status failure, the Wide IP is removed from the DNS query.

### Environment parameters

* NGINX Ingress Controller from OpenShift requires the Nginx Ingress Operator 
* BIG-IP LTM and DNS configured on the same device
* Configure BIG-IP DNS iQuery so that BIG-IP systems can communicate with each other for Data Center **ocp** and **k8s**

![iQuery](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/multi-deployment/diagram/2022-02-28_11-11-06.png)

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

cis-deployment [repo](https://github.com/mdditt2000/k8s-bigip-ctlr/tree/main/user_guides/multi-deployment-nginx/ocp/cis/cis-deployment)

**Note** Do not forget the OpenShift-SDN CNI [repo](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/multi-deployment-nginx/ocp/cni/f5-openshift-hostsubnet-01.yaml)

### Step 2: Deploy NGINX Ingress Operator on OpenShift

Recommend following [NGINX blog](https://www.nginx.com/blog/getting-started-nginx-ingress-operator-red-hat-openshift/)

nginx-config [repo](https://github.com/mdditt2000/k8s-bigip-ctlr/tree/main/user_guides/multi-deployment-nginx/ocp/nginx-config)

#### Validate NGINX Ingress Operator on OpenShift

![operator](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/multi-deployment-nginx/diagram/2022-03-02_14-33-05.png)

```
# oc get deployment -n nginx-ingress
NAME                                        READY   UP-TO-DATE   AVAILABLE   AGE
nginx-ingress-controller                    1/1     1            1           5d19h
nginx-ingress-operator-controller-manager   1/1     1            1           5d19h
[root@ocp-installer secure]# kubectl -n nginx-ingress  get all
NAME                                                            READY   STATUS    RESTARTS        AGE
pod/nginx-ingress-controller-66b4c4f7-smk8h                     1/1     Running   0               5d19h
pod/nginx-ingress-operator-controller-manager-c4fbbcb9f-xkvds   2/2     Running   1 (2d12h ago)   5d19h

NAME                                                                TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
service/nginx-ingress-controller                                    NodePort    172.30.175.208   <none>        80:30046/TCP,443:32028/TCP   5d19h
service/nginx-ingress-operator-controller-manager-metrics-service   ClusterIP   172.30.0.114     <none>        8443/TCP                     5d19h

NAME                                                        READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/nginx-ingress-controller                    1/1     1            1           5d19h
deployment.apps/nginx-ingress-operator-controller-manager   1/1     1            1           5d19h

NAME                                                                  DESIRED   CURRENT   READY   AGE
replicaset.apps/nginx-ingress-controller-66b4c4f7                     1         1         1       5d19h
replicaset.apps/nginx-ingress-operator-controller-manager-c4fbbcb9f   1         1         1       5d19h

```

## Kubernetes Container Environments

### Step 3: Deploy CIS

The benefit for using CRDs is no limitations on how many Public IPs **Virtual Server** create on BIG-IP

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

cis-deployment [repo](https://github.com/mdditt2000/k8s-bigip-ctlr/tree/main/user_guides/multi-deployment-nginx/k8s/cis/cis-deployment)

**Note** Do not forget the Flannel CNI [repo](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/multi-deployment-nginx/k8s/cni/f5-bigip-node.yaml)

### Step 4: Deploy NGINX Ingress Operator on Kubernetes

Deploy the following manifest and schema

nginx-config [repo](https://github.com/mdditt2000/k8s-bigip-ctlr/tree/main/user_guides/multi-deployment-nginx/k8s/nginx-config)

## Creating Custom Resource Definitions

### Step 5: Deploy the Cafe application

OpenShift deployment is updated to the support the API changes in 1.22

ingress-example [repo](https://github.com/mdditt2000/k8s-bigip-ctlr/tree/main/user_guides/multi-deployment-nginx/ocp/ingress-example)

Kubernetes deployment is still using beta API for IngressClass

ingress-example [repo](https://github.com/mdditt2000/k8s-bigip-ctlr/tree/main/user_guides/multi-deployment-nginx/k8s/ingress-example)

### Step 6: Creating VirtualServer, TLS and ExternalDNS CRDs using OpenShift

Use-case for the CRDs:

- Secure
- HTTP to HTTPS redirect
- Re-encrypt TLS between Client and NGINX Ingress Controller
- HTTPS Health monitor of the backend application using HOST **cafe.example.com** and **PATH /coffee, and /tea**
- Same Hostname **cafe.example.com** for OpenShift and Kubernetes environments

Diagram below displays the example of **vs-tea** with the **edns-cafe** for the following use-case

![crd-ocp](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/multi-deployment-nginx/diagram/2022-03-08_11-08-49.png)

#### Create CRDs Schema in OpenShift

**Note:** CIS requires the CustomResourceDefinition schema

```
oc create -f CustomResourceDefinition.yaml
```

CRD Schema [repo](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/multi-deployment-nginx/ocp/cis/cafe/cis-crd-schema/customresourcedefinitions.yml)

#### Create VirtualServer, TLSProfile and ExternalDNS CRDs in OpenShift

```
oc create -f reencrypt-cafe.yaml
oc create -f vs-tea.yaml
oc create -f vs-coffee.yaml
oc create -f edns-cafe.yaml
```

CRD [repo](https://github.com/mdditt2000/k8s-bigip-ctlr/tree/main/user_guides/multi-deployment-nginx/ocp/cis/cafe/secure)

#### Validate CRD

**Note** Sadly OpenShift does not have the same Dashboard for CRDs. Therefore you need to use the OpenShift CLI

```
# oc get crd,vs,tlsprofile,externaldns -n nginx-ingress
NAME                                 HOST               TLSPROFILENAME   HTTPTRAFFIC   IPADDRESS        IPAMLABEL   IPAMVSADDRESS   STATUS   AGE
virtualserver.cis.f5.com/vs-coffee   cafe.example.com   reencrypt-cafe   redirect      10.192.125.121               None            Ok       5d
virtualserver.cis.f5.com/vs-tea      cafe.example.com   reencrypt-cafe   redirect      10.192.125.121               None            Ok       5d

NAME                                   AGE
tlsprofile.cis.f5.com/reencrypt-cafe   5d19h

NAME                               DOMAINNAME         AGE   CREATED ON
externaldns.cis.f5.com/edns-cafe   cafe.example.com   5d    2022-03-03T18:31:29Z
```

#### Validate CRD using the BIG-IP

![big-ip CRD](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/multi-deployment-nginx/diagram/2022-03-08_11-16-43.png)

#### Validate CRD policy for cafe.example.com on BIG-IP

![big-ip pools](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/multi-deployment-nginx/diagram/2022-03-08_11-17-14.png)

### Step 7: Creating VirtualServer, TLS and ExternalDNS CRDs using Kubernetes

Use-case for the CRDs:

- Secure
- HTTP to HTTPS redirect
- Re-encrypt TLS between Client and NGINX Ingress Controller
- HTTPS Health monitor of the backend application using HOST **cafe.example.com** and **PATH /coffee, and /tea**
- Same Hostname **cafe.example.com** for OpenShift and Kubernetes environments

Diagram below displays the example of **vs-tea** with the **edns-cafe** for the following use-case

![crd-k8s](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/multi-deployment-nginx/diagram/2022-03-08_11-09-16.png)

#### Create CRDs Schema in Kubernetes

**Note:** CIS requires the CustomResourceDefinition schema

```
kubectl create -f CustomResourceDefinition.yaml
```

CRD Schema [repo](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/multi-deployment-nginx/k8s/cis/cafe/cis-crd-schema/customresourcedefinitions.yml)

#### Create VirtualServer and ExternalDNS CRDs in Kubernetes

```
oc create -f reencrypt-cafe.yaml
oc create -f vs-tea.yaml
oc create -f vs-coffee.yaml
oc create -f edns-cafe.yaml
```

CRD [repo](https://github.com/mdditt2000/k8s-bigip-ctlr/tree/main/user_guides/multi-deployment-nginx/k8s/cis/cafe/secure)

#### Validate CRD

```
❯ kubectl get crd,vs,tlsprofile,externaldns -n nginx-ingress
NAME                                 HOST               TLSPROFILENAME   HTTPTRAFFIC   IPADDRESS       IPAMLABEL   IPAMVSADDRESS   STATUS   AGE
virtualserver.cis.f5.com/vs-coffee   cafe.example.com   reencrypt-cafe   redirect      10.192.75.121               None            Ok       5d2h
virtualserver.cis.f5.com/vs-tea      cafe.example.com   reencrypt-cafe   redirect      10.192.75.121               None            Ok       5d2h

NAME                                   AGE
tlsprofile.cis.f5.com/reencrypt-cafe   5d22h

NAME                               DOMAINNAME         AGE    CREATED ON
externaldns.cis.f5.com/edns-cafe   cafe.example.com   5d2h   2022-03-03T18:27:10Z
```

#### Validate CRD using the BIG-IP

![big-ip CRD](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/multi-deployment-nginx/diagram/2022-03-08_12-43-28.png)

#### Validate the BIG-IP GLSB Pools

![big-ip pools](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/multi-deployment-nginx/diagram/2022-03-08_12-43-46.png)

### Step 8: Validate the BIG-IP Wide IPs and DNS Failover

#### Validate the BIG-IP GSLB Wide IP

![wide-ip](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/multi-deployment-nginx/diagram/2022-03-08_12-45-58.png)

#### Validate the BIG-IP GLSB Pools

Each pool represents a container environment. In this user-guide we have a **ocp** and **k8s** pools

![wide-ip](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/multi-deployment-nginx/diagram/2022-03-08_12-48-19.png)

#### Validate the **ocp** Data Center GLSB Pool on BIG-IP

![wide-ip](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/multi-deployment-nginx/diagram/2022-03-08_12-52-47.png)

**Note** availability shows green. Monitors are able to successfully complete the health checks

#### Validate the **ocp** Data Center GLSB Pool answer on BIG-IP

This is the public IP returned by the BIG-IP DNS to the clients DNS query

![wide-ip](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/multi-deployment-nginx/diagram/2022-03-08_12-53-05.png)

#### Validate the **k8s** Data Center GLSB Pool on BIG-IP

![wide-ip](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/multi-deployment-nginx/diagram/2022-03-08_12-51-40.png)

**Note** availability shows green. Monitors are able to successfully complete the health checks

#### Validate the **k8s** Data Center GLSB Pool answer on BIG-IP

This is the public IP returned by the BIG-IP DNS to the clients DNS query

![wide-ip](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/multi-deployment-nginx/diagram/2022-03-08_12-52-00.png)

#### Replica the **k8s** pods to zero

Replica the nginx-ingress pods in the Kubernetes container environment to zero. BIG-IP DNS will detect the replica and remove the **k8s** Data Center from the WIDE IP. Only **ocp** will be available

```
kubectl scale --current-replicas=2 --replicas=0 deployment/nginx-ingress -n nginx-ingress
```

![wide-ip](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/multi-deployment-nginx/diagram/2022-03-08_13-00-25.png)

#### Connect to the Public IP

**Note** All traffic will connect to the **ocp** cluster. I can verify this by the server address

![wide-ip](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/multi-deployment-nginx/diagram/2022-03-08_13-04-13.png)