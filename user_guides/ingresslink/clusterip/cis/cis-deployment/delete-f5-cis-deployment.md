#!/bin/bash

#delete kubernetes cis container, authentication and RBAC 
kubectl delete node bigip1
kubectl delete deployment k8s-bigip-ctlr-deployment -n kube-system
kubectl delete -f bigip-ctlr-clusterrole.yaml
kubectl delete secret bigip-login -n kube-system