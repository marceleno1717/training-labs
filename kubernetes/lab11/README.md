# Lab 11: Namespace Management and Resource Quota Enforcement

### 1) Create Namespace
Create the `ivolve` namespace to isolate application resources:
```bash
kubectl create namespace ivolve
```

### 2) Create ResourceQuota
Create `quota.yaml` to enforce a limit of 3 pods in the `ivolve` namespace:
```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: ivolvequota
  namespace: ivolve
spec:
  hard:
    pods: "3"
```

Apply the ResourceQuota:
```bash
kubectl apply -f quota.yaml
```

*(Alternatively, create it directly via CLI command):*
```bash
kubectl create quota ivolvequota --hard=pods=3 -n ivolve
```

### 3) Verify ResourceQuota Applied
Describe the ResourceQuota to check the configured pod limits and current usage:
```bash
kubectl describe resourcequota ivolvequota -n ivolve
```
```
Name:       ivolvequota
Namespace:  ivolve
Resource    Used  Hard
--------    ----  ----
pods        0     3
```

### 4) Verify Enforcement and Pod Creation Limits
Test enforcement by creating a Deployment with 3 replicas:
```bash
kubectl create deployment nginx-test --image=nginx --replicas=3 -n ivolve
```

Check the pods status and verify that all 3 pods are scheduled:
```bash
kubectl get pods -n ivolve
```
```
NAME                          READY   STATUS    RESTARTS   AGE
nginx-test-6765765798-8x7tz   1/1     Running   0          20s
nginx-test-6765765798-pl6d2   1/1     Running   0          20s
nginx-test-6765765798-v9qmk   1/1     Running   0          20s
```

Check the updated ResourceQuota usage (3 of 3 used):
```bash
kubectl describe resourcequota ivolvequota -n ivolve
```
```
Name:       ivolvequota
Namespace:  ivolve
Resource    Used  Hard
--------    ----  ----
pods        3     3
```

Attempt to scale up to 4 replicas:
```bash
kubectl scale deployment nginx-test --replicas=4 -n ivolve
```

Verify that Kubernetes prevents creating the 4th pod by inspecting the ReplicaSet events:
```bash
kubectl describe rs -n ivolve | grep -A 5 -i "quota"
```
```
Warning  FailedCreate  10s  replicaset-controller  Error creating: pods "nginx-test-6765765798-k8m9w" is forbidden: exceeded quota: ivolvequota, requested: pods=1, used: pods=3, limited: pods=3
```
