# Introduction
## Context
K8S needs to pull Docker image from AWS ECR private repository.
The AWS ECR login credential only valid in 12h

## Purpose
Setup a cronjob to periodically refresh AWS ECR login credential.

# Setup
## Build docker helper image
```
docker build -t aws-ecr-credential-helper .
```
Or you can use pre-built image pcloud/aws-ecr-credential-helper 

## Setup AWS credential
You need to have AWS_ACCESS_KEY_ID and AWS_SECRET_ACCESS_KEY which have access to AWS ECR.
```
kubectl create secret generic aws-credential --from-literal=AWS_ACCESS_KEY_ID=XXX --from-literal=AWS_SECRET_ACCESS_KEY=YYY
```

## Create K8S CronJob
Before create cronjob you need to setup environment variables in aws-ecr-credential-helper.yaml. Replace default values with your values.
```
AWS_ACCOUNT_ID
AWS_REGION
```

```
kubectl create -f aws-ecr-credential-helper.yaml
```
If your region is ap-southeast-1 then the cronjob will create a secret named ap-southeast-1-ecr-registry which contains login credential of AWS ECR. 
```
$ kubectl describe secret/ap-southeast-1-ecr-registry
Name:         ap-southeast-1-ecr-registry
Namespace:    default
Labels:       <none>
Annotations:  <none>

Type:  kubernetes.io/dockerconfigjson

Data
====
.dockerconfigjson:  4610 bytes
```

## Update your K8S deployments to pull images from AWS ECR private repo
Add imagePullSecrets in Deployment.spec.template.spec
```
imagePullSecrets:
- name: ap-southeast-1-ecr-registry
```

Example:
```
# Given a K8S deployment in:
#   AWS_ACCOUNT_ID = 123456789123
#   AWS_REGION = ap-southeast-1

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demo-deployment
  labels:
    app: demo
spec:
  selector:
    matchLabels:
      app: demo
      tier: app
      stage: dev
  template:
    metadata:
      labels:
        app: demo
        tier: app
        stage: dev
    spec:
      containers:
      - name: demo
        image: 123456789123.dkr.ecr.ap-southeast-1.amazonaws.com/demo
        ports:
        - containerPort: 8080
      imagePullSecrets:
      - name: ap-southeast-1-ecr-registry
```

Happy hacking K8S!