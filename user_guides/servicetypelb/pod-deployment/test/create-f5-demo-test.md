#!/bin/bash

#create container f5-demo-test
kubectl create -f f5-demo-test-deployment.yaml
kubectl create -f f5-demo-test-service.yaml