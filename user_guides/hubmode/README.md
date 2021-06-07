# HubMode option when using ConfigMap

**Note Preview only until CIS 2.5 release in Mid July**

HubMode expands on current ConfigMap implementation in CIS using the AS3 API. One of key strength of CIS is it can help multiple team collaborate better together. With microservices architecture we are seeing organizations create dedicate team that combine network and system personas. This allegement request that network engineers (NetOps) be added to these teams. F5 CIS ConfigMap is a perfect fit as it can accelerates the developer experience and workflow tooling.
 
Looking at the diagram below, we see the current Kubernetes personas (left in the picture) and “introduction of Operators” (the right), where different personas are responsible for configuration of the platform. Infrastructure Provider responsible for Ingress "ConfigMap" while Cluster Operator and or Application Developers responsible for Pod Deployments/Services. These personas are mostly working in different projects and or namespaces. CIS maps perfectly to both personas with the introduction on HubMode using ConfigMaps. With HubMode CIS can better assist NetOps transform from imperative to declarative using AS3 declarations, no matter of the mode. 

![diagram](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/hubmode/diagram/2021-06-07_12-13-34.png)

CIS maps perfectly to both roles. With HubMode CIS can better assist NetOps transform from imperative to declarative using AS3 declarations, no matter of the mode. In this example the Infrastructure Provider is creating the ConfigMap in the default namespace while the Application Developers is creating the Pod Deployments/Services in n1 namespace. Using HubMode, CIS will detect the associated endpoints using the service discovery labels regards of the namespace. This is consistent with CIS 1.x. 

![diagram](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/hubmode/diagram/2021-06-07_13-28-57.png)

Demo on YouTube [video]()

## Example files from the diagram:

CIS deployment arguments

```
args: 
  - "--bigip-username=$(BIGIP_USERNAME)"
  - "--bigip-password=$(BIGIP_PASSWORD)"
  - "--bigip-url=192.168.200.60"
  - "--bigip-partition=k8s"
  - "--pool-member-type=cluster"
  - "--flannel-name=fl-vxlan"
  - "--log-level=DEBUG"
  - "--insecure=true"
  - "--manage-configmaps=true"
```

* cis-deployment [repo](https://github.com/mdditt2000/k8s-bigip-ctlr/tree/main/user_guides/hubmode/cis-deployment)

ConfigMap for A1 and A2. ConfigMap applied in namespace default. Add the **hubMode: "true"** label

```
kind: ConfigMap
apiVersion: v1
metadata:
  name: f5-as3-declaration
  namespace: default
  labels:
    f5type: virtual-server
    as3: "true"
    hubMode: "true"
data:
  template: |
    {
        "class": "AS3",
        "declaration": {
            "class": "ADC",
            "schemaVersion": "3.13.0",
            "id": "urn:uuid:33045210",
            "label": "http",
            "remark": "A1 Template",
            "hubMode": {
                "class": "Tenant",
                "A1": {
                    "class": "Application",
                    "template": "generic",
                    "a1_80_vs_n1": {
                        "class": "Service_HTTP",
                        "remark": "n1 namespace",
                        "virtualAddresses": [
                            "10.192.75.101"
                        ],
                        "pool": "web_pool_n1"
                    },
                    "web_pool_n1": {
                        "class": "Pool",
                        "monitors": [
                            "http"
                        ],
                        "members": [
                            {
                                "servicePort": 8080,
                                "serverAddresses": []
                            }
                        ]
                    }
                },
                "A2": {
                    "class": "Application",
                    "template": "generic",
                    "a1_80_vs_n2": {
                        "class": "Service_HTTP",
                        "remark": "n2 namespace",
                        "virtualAddresses": [
                            "10.192.75.102"
                        ],
                        "pool": "web_pool_n2"
                    },
                    "web_pool_n2": {
                        "class": "Pool",
                        "monitors": [
                            "http"
                        ],
                        "members": [
                            {
                                "servicePort": 8080,
                                "serverAddresses": []
                            }
                        ]
                    }
                }
            }
        }
    }
```

* configmap [repo](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/hubmode/configmap/vs-configmap.yaml)

Service for endpoint A1 and A2 with the appropriate service discovery labels for CIS

```
apiVersion: v1
kind: Service
metadata:
  name: f5-hello-world-n1
  namespace: n1
  labels:
    app: f5-hello-world
    cis.f5.com/as3-tenant: hubMode
    cis.f5.com/as3-app: A1
    cis.f5.com/as3-pool: web_pool_n1
spec:
  ports:
  - name: f5-hello-world
    port: 8080
    protocol: TCP
    targetPort: 8080
  type: ClusterIP
  selector:
    app: f5-hello-world
---
apiVersion: v1
kind: Service
metadata:
  name: f5-hello-world-n1
  namespace: n2
  labels:
    app: f5-hello-world
    cis.f5.com/as3-tenant: hubMode
    cis.f5.com/as3-app: A2
    cis.f5.com/as3-pool: web_pool_n2
spec:
  ports:
  - name: f5-hello-world
    port: 8080
    protocol: TCP
    targetPort: 8080
  type: ClusterIP
  selector:
    app: f5-hello-world
```

* pod-deployment [repo](https://github.com/mdditt2000/k8s-bigip-ctlr/tree/main/user_guides/hubmode/pod-deployment)