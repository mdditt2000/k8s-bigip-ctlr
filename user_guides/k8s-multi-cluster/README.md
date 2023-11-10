# Kubernetes Multi-Cluster
Multi-cluster Kubernetes is a Kubernetes deployment method that consists of two or more clusters. This deployment method is highly flexible. You can have clusters on the same physical host or on different hosts in the same data center.

Although you can run an entire application on a single cluster, that is simply not enough in many cases. By using a multi-cluster Kubernetes architecture, you take advantage of the best this technology offers. This demo covers the basics of multi-cluster Kubernetes architecture, its benefits and challenges, and why you should consider adopting this deployment method.

Demo on YouTube [video](https://youtu.be/4Z8AVZfcNLs)

![diagram](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/k8s-multi-cluster/diagram/2023-11-01_11-01-58.png)

To achieve Multi-cluster Kubernetes, Calico CNI is required

* Configuring Calico [clouddocs](https://clouddocs.f5.com/containers/latest/userguide/calico-config.html)
* Lab Guide [lab](https://clouddocs.f5.com/training/community/containers/html/appendix/appendix8/appendix8.html#install-calico)

**Notes**

* BIG-IP needs to add all the peers: the other BIG-IP, our k8s nodes
