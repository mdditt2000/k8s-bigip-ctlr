## Deploying F5 Container Ingress Services (CIS) in Rancher using Helm Charts

Using chart simplifies repeatable, versioned deployment of the CIS. This is the simplest way to install the CIS on Rancher cluster. Helm is a package manager for Kubernetes. Helm is Kubernetes version of yum or apt. Helm deploys something called charts, which you can think of as a packaged application. It is a collection of all your versioned, pre-configured application resources which can be deployed as one unit. This chart creates a Deployment for one Pod containing the k8s-bigip-ctlr, it's supporting RBAC, Service Account and Custom Resources Definition installations. The purpose of this user-guide is demonstrate the simplicity of CIS with Rancher using Helm Charts with custom-resources.

Demo on YouTube [video]()simple

### Installing the Chart for the F5 CIS

* Add BIG-IP credentials as K8S secrets

    kubectl create secret generic f5-bigip-ctlr-login -n kube-system --from-literal=username=admin --from-literal=password=<secret>

* Add the CIS chart repository in Helm using following command:

    helm repo add f5-stable https://f5networks.github.io/charts/stable

* Install the Helm chart using the following command:

    helm install -f https://raw.githubusercontent.com/mdditt2000/rancher/main/cis-deployment/cis-k8s-custom-resource-values.yaml f5-bigip-ctlr f5-stable/f5-bigip-ctlr

#### Creating the CIS deployment from the Helm Chart

```
> kubectl create secret generic f5-bigip-ctlr-login -n kube-system --from-literal=username=admin --from-literal=password=<secret>
secret/f5-bigip-ctlr-login created

> helm repo add f5-stable https://f5networks.github.io/charts/stable
"f5-stable" has been added to your repositories

> helm install -f https://raw.githubusercontent.com/mdditt2000/rancher/main/cis-deployment/cis-k8s-custom-resource-values.yaml f5-bigip-ctlr f5-stable/f5-bigip-ctlr
creating 5 resource(s)
beginning wait for 5 resources with timeout of 1m0s
creating 5 resource(s)
NAME: f5-bigip-ctlr
LAST DEPLOYED: Wed Nov 10 20:57:21 2021
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
F5 BIG-IP controller: f5-bigip-ctlr

General Controller Documentation:
- Kubernetes: http://clouddocs.f5.com/containers/latest/kubernetes/index.html
- OpenShift: http://clouddocs.f5.com/containers/latest/openshift/index.html

Using Ingress? There's a helm chart for that:
- https://github.com/F5Networks/charts/tree/master/src/stable/f5-bigip-ingress

Using Routes in OpenShift? No helm chart yet, but we do have great documentation:
- http://clouddocs.f5.com/containers/latest/openshift/kctlr-openshift-routes.html
```

View installed Apps from the Rancher dashboard as shown in the example

![installed-apps](https://github.com/mdditt2000/rancher/blob/main/diagrams/2021-11-10_13-10-48.png)

Validate that CIS is deployed and running from the Deployment tab in the Rancher Dashboard

![validate](https://github.com/mdditt2000/rancher/blob/main/diagrams/2021-11-10_13-19-00.png)

Select the **Deployment: f5-bigip-ctlr** from the Deployment tab in the Rancher Dashboard

![validate](https://github.com/mdditt2000/rancher/blob/main/diagrams/2021-11-10_13-19-00.png)

**Congratulations CIS is deployed in Rancher**