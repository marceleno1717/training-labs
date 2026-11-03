# Lab 19: Securing Kubernetes with RBAC and Service Accounts

### 1) Create Service Account (jenkins-sa.yaml)
Create `jenkins-sa.yaml` defining the `jenkins-sa` ServiceAccount in the `ivolve` namespace:
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: jenkins-sa
  namespace: ivolve
```

Apply the ServiceAccount manifest:
```bash
kubectl apply -f jenkins-sa.yaml
```

Verify ServiceAccount creation:
```bash
kubectl get sa -n ivolve
```
```
NAME         SECRETS   AGE
default      0         6d23h
jenkins-sa   0         3h21m
```

---

### 2) Create Service Account Token

#### Method A: Imperative Token Generation (CLI)
Generate a short-lived token:
```bash
kubectl create token jenkins-sa -n ivolve
```

Generate a token with custom duration (e.g., 24 hours):
```bash
kubectl create token jenkins-sa -n ivolve --duration=24h -o yaml
```

#### Method B: Long-Lived Secret Token Manifest (secret.yaml)
Create `secret.yaml` with the `kubernetes.io/service-account.name` annotation:
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: jenkins-sa-token
  namespace: ivolve
  annotations:
    kubernetes.io/service-account.name: jenkins-sa
type: kubernetes.io/service-account-token
```

Apply the Secret:
```bash
kubectl apply -f secret.yaml
```

---

### 3) Define Role Manifest (role.yaml)
Create `role.yaml` defining `ivolveDeployRole` in the `ivolve` namespace granting permissions on `pods`:
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: ivolve
  name: ivolveDeployRole
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "create", "update"]
```

Apply the Role:
```bash
kubectl apply -f role.yaml
```

---

### 4) Create RoleBinding Manifest (rolebinding.yaml)
Create `rolebinding.yaml` binding `ivolveDeployRole` to `jenkins-sa` in the `ivolve` namespace:
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: ivolveDeployRoleBinding
  namespace: ivolve
subjects:
  - kind: ServiceAccount
    name: jenkins-sa
    namespace: ivolve
roleRef:
  kind: Role
  name: ivolveDeployRole
  apiGroup: rbac.authorization.k8s.io
```

Apply the RoleBinding:
```bash
kubectl apply -f rolebinding.yaml
```

---

### 5) Validate Permissions Using `kubectl auth can-i`
Validate what actions `jenkins-sa` can perform in the `ivolve` namespace:
```bash
kubectl auth can-i list pods --as=system:serviceaccount:ivolve:jenkins-sa -n ivolve
```
```
yes
```
```bash
kubectl auth can-i create pods --as=system:serviceaccount:ivolve:jenkins-sa -n ivolve
```
```
yes
```
```bash
kubectl auth can-i delete pods --as=system:serviceaccount:ivolve:jenkins-sa -n ivolve
```
```
no
```
```bash
kubectl auth can-i list pods --as=system:serviceaccount:ivolve:jenkins-sa -n default
```
```
no
```
