# Lab 12: Managing Configuration and Sensitive Data with ConfigMaps and Secrets

### 1) Create ConfigMap Manifest (configmap.yaml)
Create `configmap.yaml` in the `ivolve` namespace to store non-sensitive MySQL configuration variables:
- `DB_HOST`: Hostname of the MySQL StatefulSet's headless service (`ivolvedbsvc`).
- `DB_USER`: Application database user (`admin`).

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: ivolvecm
  namespace: ivolve
data:
  DB_HOST: "ivolvedbsvc"
  DB_USER: "admin"
```

Apply the ConfigMap:
```bash
kubectl apply -f configmap.yaml
```

*(Alternatively, create it directly via CLI command):*
```bash
kubectl create configmap ivolvecm --from-literal=DB_HOST=ivolvedbsvc --from-literal=DB_USER=admin -n ivolve
```

### 2) Encode Sensitive Credentials using Base64
Generate the Base64 encoded values for the MySQL secrets (`MYSQL_ROOT_PASSWORD` = `123`, `DB_PASSWORD` = `admin123`):
```bash
echo -n '123' | base64
```
```
MTIz
```
```bash
echo -n 'admin123' | base64
```
```
YWRtaW4xMjM=
```

### 3) Create Secret Manifest (secret.yaml)
Create `secret.yaml` in the `ivolve` namespace using the Base64 encoded values:
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: ivolvesecret
  namespace: ivolve
type: Opaque
data:
  MYSQL_ROOT_PASSWORD: MTIz
  DB_PASSWORD: YWRtaW4xMjM=
```

Apply the Secret:
```bash
kubectl apply -f secret.yaml
```

*(Alternatively, create it directly via CLI command):*
```bash
kubectl create secret generic ivolvesecret --from-literal=MYSQL_ROOT_PASSWORD=123 --from-literal=DB_PASSWORD=admin123 -n ivolve
```

### 4) Verify ConfigMap
Verify the ConfigMap creation and inspect its stored keys:
```bash
kubectl get configmap ivolvecm -n ivolve
```
```
NAME       DATA   AGE
ivolvecm   2      1m
```
```bash
kubectl describe configmap ivolvecm -n ivolve
```
```
Name:         ivolvecm
Namespace:    ivolve
Labels:       <none>
Annotations:  <none>

Data
====
DB_HOST:
----
ivolvedbsvc
DB_USER:
----
admin
```

### 5) Verify Secret
Verify the Secret creation and keys:
```bash
kubectl get secret ivolvesecret -n ivolve
```
```
NAME           TYPE     DATA   AGE
ivolvesecret   Opaque   2      1m
```
```bash
kubectl describe secret ivolvesecret -n ivolve
```
```
Name:         ivolvesecret
Namespace:    ivolve
Labels:       <none>
Annotations:  <none>

Type:  Opaque

Data
====
DB_PASSWORD:          8 bytes
MYSQL_ROOT_PASSWORD:  3 bytes
```

Verify decoding the Secret values:
```bash
kubectl get secret ivolvesecret -n ivolve -o jsonpath="{.data.DB_PASSWORD}" | base64 --decode && echo ""
```
```
admin123
```
```bash
kubectl get secret ivolvesecret -n ivolve -o jsonpath="{.data.MYSQL_ROOT_PASSWORD}" | base64 --decode && echo ""
```
```
123
```
