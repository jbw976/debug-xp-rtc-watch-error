# Repro Project for Crossplane Realtime Compositions Watch Errors

This repo contains repro steps and manifests for debugging watch errors in Crossplane realtime compositions, e.g., [crossplane#5957](https://github.com/crossplane/crossplane/issues/5957).

## Pre-Requisites

### Install Crossplane

Create a Kubernetes cluster, e.g. with `kind`:
```
kind create cluster
```

Install Crossplane `1.20.0-rc.1`:
```
helm repo add crossplane-stable https://charts.crossplane.io/stable --force-update
helm install crossplane --namespace crossplane-system --create-namespace crossplane-stable/crossplane --devel --version 1.20.0-rc.1 --set args='{"--debug"}'
```

### Install Providers and Functions

Install the AWS provider:
```
kubectl apply -f provider.yaml
kubectl apply -f functions.yaml
```

Wait for the functions and AWS providers to become installed and healthy:
```
kubectl get pkg
```

Create credentials for the AWS provider to create resources in your AWS account:
```
AWS_PROFILE=default && echo -e "[default]\naws_access_key_id = $(aws configure get aws_access_key_id --profile $AWS_PROFILE)\naws_secret_access_key = $(aws configure get aws_secret_access_key --profile $AWS_PROFILE)" > aws-creds.txt
kubectl create secret generic aws-creds -n crossplane-system --from-file=credentials=./aws-creds.txt
kubectl apply -f provider-config-default.yaml
```

## Install Repro Resources

Create an XRD and Composition for `Parent` and `Child` XRs:
```
kubectl apply -f xrd-parent.yaml
kubectl apply -f composition-parent.yaml
kubectl apply -f xrd-child.yaml
kubectl apply -f composition-child.yaml
```

## Create Resources

Create a `Parent` claim that will start a complicated reconciliation of nested composite and composed resources:
```
kubectl apply -f claim.yaml
```

Check the logs to see if the `Unhandled Error` watch error has occurred:
```
kubectl -n crossplane-system logs --tail=-1 -l app=crossplane -c crossplane | grep -F -i 'Unhandled Error'
2025-05-18T16:32:18Z    ERROR   crossplane      Unhandled Error {"logger": "UnhandledError", "error": "pkg/mod/k8s.io/client-go@v0.31.2/tools/cache/reflector.go:243: expected type *composed.Unstructured, but watch event obj
ect had type *unstructured.Unstructured"}
2025-05-18T16:33:16Z    ERROR   crossplane      Unhandled Error {"logger": "UnhandledError", "error": "pkg/mod/k8s.io/client-go@v0.31.2/tools/cache/reflector.go:243: expected type *composed.Unstructured, but watch event object had type *unstructured.Unstructured"}
...
```

Perform a `trace` of the resources to verify that they never converge to completely `Ready`:
```
crossplane beta trace parent.debug.crossplane.io/parent
NAME                                             SYNCED   READY   STATUS
Parent/parent (default)                          True     False   Waiting: Claim is waiting for composite resource to become Ready
└─ XParent/parent-t2rlj                          True     False   Creating: ...: xchild-debugging-0, xchild-debugging-1, and xchild-debugging-2
   ├─ XChild/xchild-debugging-0                  True     False   Creating: ...ugging-0-0, gateway-debugging-0-1, vpc-debugging-0-0, and 1 more
   │  ├─ InternetGateway/gateway-debugging-0-0   True     False   Creating
   │  ├─ InternetGateway/gateway-debugging-0-1   True     False   Creating
   │  ├─ VPC/vpc-debugging-0-0                   True     True    Available
   │  └─ VPC/vpc-debugging-0-1                   True     True    Available
   ├─ XChild/xchild-debugging-1                  True     False   Creating: ...ugging-1-0, gateway-debugging-1-1, vpc-debugging-1-0, and 1 more
   │  ├─ InternetGateway/gateway-debugging-1-0   True     True    Available
   │  ├─ InternetGateway/gateway-debugging-1-1   True     False   Creating
   │  ├─ VPC/vpc-debugging-1-0                   True     True    Available
   │  └─ VPC/vpc-debugging-1-1                   True     True    Available
   └─ XChild/xchild-debugging-2                  True     False   Creating: ...ugging-2-0, gateway-debugging-2-1, vpc-debugging-2-0, and 1 more
      ├─ InternetGateway/gateway-debugging-2-0   True     False   Creating
      ├─ InternetGateway/gateway-debugging-2-1   True     False   Creating
      ├─ VPC/vpc-debugging-2-0                   True     True    Available
      └─ VPC/vpc-debugging-2-1                   True     True    Available
```
