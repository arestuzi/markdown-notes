# ALB Ingress failed to launch with error "The AWS Access Key Id needs a subscription"

## Environment

- ALB Ingress Controller 2.0.0

## Issue

- Pod for ALB Ingress fails to create

## Resolution

1.Modify the deployment of ALB Ingress Controller

```bash
spec:
  containers:
  - args:
    - --cluster-name=eks
    - --ingress-class=alb
    - --enable-shield=false
    - --enable-waf=false
    - --enable-wafv2=false
```

## Root Cause

- AWS Shield and AWS WAF do not launch in China region and latest version of ALB Ingress Controller supports these supports which cause this issue.

## Diagnose

```bash
kubectl logs pod aws-load-balancer-controller-5694dc97cf-wcwr9 -n kube-system

FailedDeployModel
Failed deploy model due to SubscriptionRequiredException: The AWS Access Key Id needs a subscription for the service status code: 400, request id: dc8b5fe5-8946-4596-9521-15bf946a6fc7

FailedDeployModel
Failed deploy model due to SubscriptionRequiredException: The AWS Access Key Id needs a subscription for the service status code: 400, request id: 66d3979a-8647-4b14-9982-44f79b6bcbf1
```
