#!/bin/bash

#delete ipam controller authentication RBAC
kubectl delete -f f5-ipam-rbac.yaml
kubectl delete -f f5-ipam-schema.yaml
kubectl delete -f f5-ipam-persitentvolume.yaml
kubectl delete -f f5-ipam-deployment.yaml