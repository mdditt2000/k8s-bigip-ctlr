### Get Node for OpenShift 4-11

```
[root@ocp-installer cis]# oc get nodes
NAME                        STATUS   ROLES    AGE    VERSION
ocp-pm-trw88-master-0       Ready    master   254d   v1.24.0+3882f8f
ocp-pm-trw88-master-1       Ready    master   254d   v1.24.0+3882f8f
ocp-pm-trw88-master-2       Ready    master   254d   v1.24.0+3882f8f
ocp-pm-trw88-worker-d6zsg   Ready    worker   254d   v1.24.0+3882f8f
ocp-pm-trw88-worker-k7lsd   Ready    worker   254d   v1.24.0+3882f8f
ocp-pm-trw88-worker-vdtmb   Ready    worker   254d   v1.24.0+3882f8f
```

#### Workers
```
# oc describe node ocp-pm-trw88-worker-d6zsg |grep "node-subnets\|node-primary-ifaddr"
k8s.ovn.org/node-primary-ifaddr: {"ipv4":"10.192.125.172/24"}
k8s.ovn.org/node-subnets: {"default":"10.129.2.0/23"}
```

```
# oc describe node ocp-pm-trw88-worker-k7lsd |grep "node-subnets\|node-primary-ifaddr"
k8s.ovn.org/node-primary-ifaddr: {"ipv4":"10.192.125.174/24"}
k8s.ovn.org/node-subnets: {"default":"10.131.0.0/23"}
```

```
# oc describe node ocp-pm-trw88-worker-vdtmb |grep "node-subnets\|node-primary-ifaddr"
k8s.ovn.org/node-primary-ifaddr: {"ipv4":"10.192.125.177/24"}
k8s.ovn.org/node-subnets: {"default":"10.128.2.0/23"}
```
Static Routes on BIG-IP

tmsh create /net route 10.129.2.0/23 gw 10.192.125.172
tmsh create /net route 10.131.0.0/23 gw 10.192.125.174
tmsh create /net route 10.128.2.0/23 gw 10.192.125.177

---

### Get Node for OpenShift 4-13

```
# oc get nodes
NAME                           STATUS   ROLES                  AGE   VERSION
ocp-tpm-zrs4x-master-0         Ready    control-plane,master   86m   v1.26.3+b404935
ocp-tpm-zrs4x-master-1         Ready    control-plane,master   86m   v1.26.3+b404935
ocp-tpm-zrs4x-master-2         Ready    control-plane,master   87m   v1.26.3+b404935
ocp-tpm-zrs4x-worker-0-cn5rh   Ready    worker                 68m   v1.26.3+b404935
ocp-tpm-zrs4x-worker-0-gb76z   Ready    worker                 66m   v1.26.3+b404935
ocp-tpm-zrs4x-worker-0-n8tn4   Ready    worker                 69m   v1.26.3+b404935

```

#### Workers

```
# oc describe node ocp-tpm-zrs4x-worker-0-cn5rh |grep "node-subnets\|node-primary-ifaddr"
k8s.ovn.org/node-primary-ifaddr: {"ipv4":"10.192.125.168/24"}
k8s.ovn.org/node-subnets: {"default":["10.148.2.0/23"]}
```

```
# oc describe node ocp-tpm-zrs4x-worker-0-gb76z |grep "node-subnets\|node-primary-ifaddr"
k8s.ovn.org/node-primary-ifaddr: {"ipv4":"10.192.125.173/24"}
k8s.ovn.org/node-subnets: {"default":["10.149.2.0/23"]}
```

```
# oc describe node ocp-tpm-zrs4x-worker-0-n8tn4 |grep "node-subnets\|node-primary-ifaddr"
k8s.ovn.org/node-primary-ifaddr: {"ipv4":"10.192.125.167/24"}
k8s.ovn.org/node-subnets: {"default":["10.151.0.0/23"]}
```

Static Routes on BIG-IP

tmsh create /net route 10.148.2.0/23 gw 10.192.125.168
tmsh create /net route 10.149.2.0/23 gw 10.192.125.173
tmsh create /net route 10.151.0.0/23 gw 10.192.125.167