# Lab 17: Control Pod-to-Pod Traffic via Network Policy

### 1) Create NetworkPolicy Manifest (networkpolicy.yaml)
Create `networkpolicy.yaml` in the `ivolve` namespace to secure MySQL traffic:
- **Name**: `ivolvenp`
- **Pod Selector**: Targets the MySQL pod (`app: ivolvedb`).
- **Policy Types**: Only `Ingress`.
- **Ingress Rule**:
  - Restricts inbound traffic only to pods with label `app: ivolveapp`.
  - Restricts access strictly to MySQL port `3306` (TCP).

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: ivolvenp
  namespace: ivolve
spec:
  podSelector:
    matchLabels:
      app: ivolvedb
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: ivolveapp
      ports:
        - protocol: TCP
          port: 3306
```

Apply the NetworkPolicy:
```bash
kubectl apply -f networkpolicy.yaml
```

---

### 2) Verify NetworkPolicy Creation
Verify the NetworkPolicy is applied and inspect its configuration:
```bash
kubectl get networkpolicy -n ivolve
```
```
NAME       POD-SELECTOR   AGE
ivolvenp   app=ivolvedb   1m
```
```bash
kubectl describe networkpolicy ivolvenp -n ivolve
```
```
Name:         ivolvenp
Namespace:    ivolve
Labels:       <none>
Annotations:  <none>
Spec:
  PodSelector:     app=ivolvedb
  Allowing ingress traffic:
    To Port: 3306/TCP
    From:
      PodSelector: app=ivolveapp
  Not affecting egress traffic
  Policy Types: Ingress
```

---

### 3) Verify Allowed Ingress from Application Pods (Authorized)
Test connectivity to MySQL port `3306` from an authorized `ivolveapp` application pod:
```bash
kubectl exec -it -n ivolve $(kubectl get pods -n ivolve -l app=ivolveapp -o jsonpath="{.items[0].metadata.name}") -c app -- nc -zv -w 3 ivolvedbsvc 3306
```
```
ivolvedbsvc (10.244.205.194:3306) open
```

Verify that the Node.js application continues serving traffic normally:
```bash
kubectl exec -it -n ivolve network -- curl -s http://ivolvaappsvc:3000/health
```
```
🚀 iVolve web app is working! Keep calm and code on! 🎉
```

---

### 4) Verify Blocked Ingress from Other Pods (Unauthorized)
Attempt to connect to MySQL on port `3306` from an unauthorized pod (e.g., the `network` diagnostic pod):
```bash
kubectl exec -it -n ivolve network -- nc -zv -w 3 ivolvedbsvc 3306
```
```
nc: connect to ivolvedbsvc (10.244.205.194) port 3306 (tcp) timed out: Operation in progress
command terminated with exit code 1
```

*Result: Connection is blocked and times out as enforced by the `ivolvenp` NetworkPolicy.*
