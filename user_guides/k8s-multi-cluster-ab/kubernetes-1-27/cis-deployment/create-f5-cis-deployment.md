#!/bin/bash
#create kubernetes cis container, authentication and RBAC
kubectl create secret generic bigip-login -n kube-system --from-literal=username=admin --from-literal=password=f5PME123
kubectl create -f bigip-ctlr-clusterrole.yaml
kubectl create -f f5-bigip-ctlr-deployment.yaml




