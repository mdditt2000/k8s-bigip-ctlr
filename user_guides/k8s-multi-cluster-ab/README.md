# Multi-Cluster Kubernetes using A/B Deployment

This diagram displays how to use Multi-cluster kubernetes using A/B deployments ratios to load balance the traffic across two different Kubernetes clusters. The ratio is weighted to 80% to Kubernetes 1.27 cluster and remaining to the 1.28%.

Demo on YouTube [video](https://youtu.be/hd0TIVff2Tc)

![diagram](https://github.com/mdditt2000/multi-cluster/blob/main/k8s-multi-cluster-ab/diagram/2023-11-17_09-04-51.png)

F5 BIG-IP as Kubernetes gateway will distribute the traffic to the clusters. This example is using no persistance. However depending on your application you might need to maintain persistence connections across the clusters. Default is Cookie Persistence. F5 CIS creates to Pools on BIG-IP shown below. 	Pool **coffee_svc_a_8080** for **Kubernetes 1.27** cluster and Pool **coffee_svc_b_8080** for cluster **Kubernetes 1.28**

![pool](https://github.com/mdditt2000/multi-cluster/blob/main/k8s-multi-cluster-ab/diagram/2023-11-17_09-19-19.png)

Diagram below show the weighted distribution of traffic between the clusters. **80%** more traffic to the cluster **Kubernetes 1.27**

![pool](https://github.com/mdditt2000/multi-cluster/blob/main/k8s-multi-cluster-ab/diagram/2023-11-17_09-39-12.png)

The link below explain how setup the Calico CNI for the Multi-Cluster Kubernetes environment

* Configuring Calico [clouddocs](https://clouddocs.f5.com/containers/latest/userguide/calico-config.html)
* Lab Guide [lab](https://clouddocs.f5.com/training/community/containers/html/appendix/appendix8/appendix8.html#install-calico)

**Notes**

* BIG-IP needs to add all the peers: the other BIG-IP, our k8s nodes