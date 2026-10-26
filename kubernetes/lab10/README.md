# Lab 10: Node Isolation Using Taints and Labels in Kubernetes

### 1) Apply Taint to Worker Node
Apply a taint to the worker node (`ip-172-31-8-8`) with key `workload`, value `ivolveApp`, and effect `NoSchedule` to prevent pods without matching tolerations from scheduling on it:
```bash
kubectl taint nodes ip-172-31-8-8 workload=ivolveApp:NoSchedule
```

### 2) Label Worker Node
Label the worker node (`ip-172-31-8-8`) with `project=ivolve` to identify it for targeted pod scheduling:
```bash
kubectl label nodes ip-172-31-8-8 project=ivolve
```

### 3) Verify Taint on Worker Node
Inspect the node details and verify that the taint `workload=ivolveApp:NoSchedule` is applied:
```bash
kubectl describe node ip-172-31-8-8 | grep -i taints
```
```
Taints:             workload=ivolveApp:NoSchedule
```

### 4) Verify Label on Worker Node
Verify that the label `project=ivolve` has been applied to the node:
```bash
kubectl get nodes --show-labels | grep ip-172-31-8-8
```
```
ip-172-31-8-8      Ready    <none>   1d   v1.35.9   ...,project=ivolve,...
```

Or query specifically using label selector:
```bash
kubectl get nodes -l project=ivolve
```
```
NAME            STATUS   ROLES    AGE   VERSION
ip-172-31-8-8   Ready    <none>   1d    v1.35.9
```
