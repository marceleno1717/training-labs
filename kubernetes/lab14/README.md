# Lab 14: StatefulSet with Headless Service

### 1) Copy ConfigMap and Secret from Lab 12
Copy the configuration and secret manifests:
```bash
cp ../lab12/secret.yaml ../lab12/configmap.yaml .
```

Apply both manifests in `ivolve` namespace:
```bash
kubectl apply -f configmap.yaml
```
```bash
kubectl apply -f secret.yaml
```

---

### 2) Create Headless Service Manifest (service.yaml)
Create `service.yaml` defining a headless service (`clusterIP: None`) named `ivolvedbsvc` in the `ivolve` namespace to provide stable network identity and direct DNS resolution for the StatefulSet pod:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: ivolvedbsvc
  namespace: ivolve
  labels:
    app: ivolvedb
spec:
  clusterIP: None
  ports:
    - port: 3306
      name: mysql
  selector:
    app: ivolvedb
```

Apply the Headless Service:
```bash
kubectl apply -f service.yaml
```

---

### 3) Create StatefulSet Manifest (statefulset.yaml)
Create `statefulset.yaml` defining the `ivolvedb` StatefulSet in the `ivolve` namespace:
- 1 replica running `mysql:latest`.
- Sensitive credentials (`MYSQL_ROOT_PASSWORD`, `DB_PASSWORD`) sourced from `ivolvesecret`.
- Non-sensitive user (`MYSQL_USER`) sourced from `ivolvecm`.
- Target scheduling via `nodeSelector` (`project: ivolve`).
- Matching `tolerations` for the worker node taint (`workload=ivolveApp:NoSchedule`).
- VolumeClaimTemplate dynamically mounting `/var/lib/mysql`.

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: ivolvedb
  namespace: ivolve
spec:
  serviceName: ivolvedbsvc
  replicas: 1
  selector:
    matchLabels:
      app: ivolvedb
  template:
    metadata:
      labels:
        app: ivolvedb
    spec:
      nodeSelector:
        project: ivolve
      tolerations:
        - key: workload
          operator: Equal
          value: ivolveApp
          effect: NoSchedule
      containers:
        - name: mysql
          image: mysql:latest
          ports:
            - containerPort: 3306
              name: mysql
          env:
            - name: MYSQL_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: ivolvesecret
                  key: MYSQL_ROOT_PASSWORD
            - name: MYSQL_USER
              valueFrom:
                configMapKeyRef:
                  name: ivolvecm
                  key: DB_USER
            - name: MYSQL_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: ivolvesecret
                  key: DB_PASSWORD
            - name: MYSQL_DATABASE
              value: ivolve
          volumeMounts:
            - name: mysql-data
              mountPath: /var/lib/mysql
  volumeClaimTemplates:
    - metadata:
        name: mysql-data
      spec:
        accessModes:
          - ReadWriteOnce
        resources:
          requests:
            storage: 1Gi
```

Apply the StatefulSet:
```bash
kubectl apply -f statefulset.yaml
```

---

### 4) Verify StatefulSet and Pod Creation
Verify the StatefulSet deployment and running pods:
```bash
kubectl get statefulset ivolvedb -n ivolve
```
```
NAME       READY   AGE
ivolvedb   1/1     30m
```
```bash
kubectl get pods -n ivolve -o wide
```
```
NAME         READY   STATUS    RESTARTS   AGE   IP           NODE           NOMINATED NODE   READINESS GATES
ivolvedb-0   1/1     Running   0          30m   10.244.1.7   minikube-m02   <none>           <none>
```

Verify the dynamically provisioned PersistentVolumeClaim:
```bash
kubectl get pvc -n ivolve
```
```
NAME                      STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
mysql-data-ivolvedb-0     Bound    pvc-87612f0a-012a-43d1-bf34-73349e5d7a81   1Gi        RWO            standard       30m
```

---

### 5) Verify Database Operational Status via MySQL Client
Execute an interactive MySQL client query inside the `ivolvedb-0` pod using credentials from `ivolvesecret`:
```bash
kubectl exec -it ivolvedb-0 -n ivolve -- mysql -uadmin -padmin123 -e "SHOW DATABASES;"
```
```
+--------------------+
| Database           |
+--------------------+
| information_schema |
| ivolve             |
| performance_schema |
+--------------------+
```

---

### 6) Verify MySQL Pod DNS Resolution Inside Cluster
Run a diagnostic network container (`nicolaka/netshoot`) in the `ivolve` namespace to test DNS resolution:
```bash
kubectl run -n ivolve network --image nicolaka/netshoot -it -- bash
```

Inside the `network` container, perform DNS lookup for the StatefulSet pod via the headless service:
```bash
nslookup ivolvedb-0.ivolvedbsvc
```
```
Server:         10.96.0.10
Address:        10.96.0.10#53

Name:   ivolvedb-0.ivolvedbsvc.ivolve.svc.cluster.local
Address: 10.244.1.7
```

Exit the diagnostic pod:
```bash
exit
```

Verify cluster pods status:
```bash
kubectl get pods -n ivolve -o wide
```
```
NAME         READY   STATUS    RESTARTS     AGE     IP           NODE           NOMINATED NODE   READINESS GATES
ivolvedb-0   1/1     Running   0            30m     10.244.1.7   minikube-m02   <none>           <none>
network      1/1     Running   1 (9s ago)   2m28s   10.244.0.6   minikube       <none>           <none>
```
