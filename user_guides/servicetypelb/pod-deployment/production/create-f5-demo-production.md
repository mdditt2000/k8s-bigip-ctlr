#!/bin/bash

#create container f5-demo-production
kubectl create -f f5-demo-deployment-production.yaml
kubectl create -f f5-demo-service-production.yaml