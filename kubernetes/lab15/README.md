# Lab 15: Node.js Application Deployment with ClusterIP Service

### 1) Ensure PersistentVolume and PVC Are Available
Ensure `ivolvepv` and `ivolvepvc` created in Lab 13 are applied:
```bash
kubectl apply -f ../lab13/pv.yaml && kubectl apply -f ../lab13/pvc.yaml
```

Verify storage status:
```bash
kubectl get pv ivolvepv && kubectl get pvc ivolvepvc -n ivolve
```
```
NAME       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM              STORAGECLASS   VOLUMEATTRIBUTESCLASS   REASON   AGE
ivolvepv   1Gi        RWX            Retain           Bound    ivolve/ivolvepvc                  <unset>                          1m
NAME        STATUS   VOLUME     CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
ivolvepvc   Bound    ivolvepv   1Gi        RWX                           <unset>                 1m
```

---

### 2) Create Deployment Manifest (deployment.yaml)
Create `deployment.yaml` defining the `ivolveapp` Deployment in the `ivolve` namespace with:
- `2` replicas running your custom Docker image `marceleno17/lab8-app:latest`.
- `initContainers`: Runs `mysql:latest` client to wait for MySQL, create the `ivolve` database, and grant privileges to the application user using parameters from `ivolvecm` and `ivolvesecret`.
- App container consuming `DB_HOST`, `DB_USER` from `ivolvecm` and `DB_PASSWORD` from `ivolvesecret`.
- PersistentVolume storage using `ivolvepvc` mounted to `/app/logs`.
- Targeted scheduling with `nodeSelector` (`project: ivolve`) and `tolerations` (`workload=ivolveApp:NoSchedule`).

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ivolveapp
  namespace: ivolve
spec:
  replicas: 2
  selector:
    matchLabels:
      app: ivolveapp
  template:
    metadata:
      labels:
        app: ivolveapp
    spec:
      nodeSelector:
        project: ivolve
      tolerations:
        - key: workload
          operator: Equal
          value: ivolveApp
          effect: NoSchedule
      initContainers:
        - name: init-db
          image: mysql:latest
          env:
            - name: DB_HOST
              valueFrom:
                configMapKeyRef:
                  name: ivolvecm
                  key: DB_HOST
            - name: MYSQL_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: ivolvesecret
                  key: MYSQL_ROOT_PASSWORD
            - name: DB_USER
              valueFrom:
                configMapKeyRef:
                  name: ivolvecm
                  key: DB_USER
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: ivolvesecret
                  key: DB_PASSWORD
          command:
            - "sh"
            - "-c"
            - |
              until mysql -h "$DB_HOST" -u root -p"$MYSQL_ROOT_PASSWORD" -e "SELECT 1;" > /dev/null 2>&1; do
                echo "Waiting for MySQL at $DB_HOST to be ready..."
                sleep 3
              done
              mysql -h "$DB_HOST" -u root -p"$MYSQL_ROOT_PASSWORD" -e "CREATE DATABASE IF NOT EXISTS ivolve; CREATE USER IF NOT EXISTS '$DB_USER'@'%' IDENTIFIED BY '$DB_PASSWORD'; GRANT ALL PRIVILEGES ON ivolve.* TO '$DB_USER'@'%'; FLUSH PRIVILEGES;"
              echo "Database ivolve initialized and privileges granted to $DB_USER."
      containers:
        - name: app
          image: marceleno17/lab8-app:latest
          ports:
            - containerPort: 3000
              name: http
          env:
            - name: DB_HOST
              valueFrom:
                configMapKeyRef:
                  name: ivolvecm
                  key: DB_HOST
            - name: DB_USER
              valueFrom:
                configMapKeyRef:
                  name: ivolvecm
                  key: DB_USER
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: ivolvesecret
                  key: DB_PASSWORD
          volumeMounts:
            - name: log-volume
              mountPath: /app/logs
      volumes:
        - name: log-volume
          persistentVolumeClaim:
            claimName: ivolvepvc
```

Apply the Deployment:
```bash
kubectl apply -f deployment.yaml
```

---

### 3) Create ClusterIP Service Manifest (service.yaml)
Create `service.yaml` defining a ClusterIP service named `ivolvaappsvc` in `ivolve` namespace to distribute HTTP traffic across the application pods on port 3000:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: ivolvaappsvc
  namespace: ivolve
  labels:
    app: ivolveapp
spec:
  type: ClusterIP
  ports:
    - port: 3000
      targetPort: 3000
      name: http
  selector:
    app: ivolveapp
```

Apply the ClusterIP Service:
```bash
kubectl apply -f service.yaml
```

---

### 4) Verify Deployment and Pod Status
Verify the Deployment and running replicas:
```bash
kubectl get deployment ivolveapp -n ivolve
```
```
NAME        READY   UP-TO-DATE   AVAILABLE   AGE
ivolveapp   2/2     2            2           94s
```
```bash
kubectl get pods -l app=ivolveapp -n ivolve -o wide
```
```
NAME                         READY   STATUS    RESTARTS   AGE   IP           NODE           NOMINATED NODE   READINESS GATES
ivolveapp-5464f575d4-mg4pb   1/1     Running   0          67s   10.244.1.8   minikube-m02   <none>           <none>
ivolveapp-5464f575d4-s6n8f   1/1     Running   0          67s   10.244.1.9   minikube-m02   <none>           <none>
```

Verify ClusterIP Service endpoints:
```bash
kubectl get svc ivolvaappsvc -n ivolve
```
```
NAME           TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
ivolvaappsvc   ClusterIP   10.98.243.216   <none>        3000/TCP   94s
```
```bash
kubectl get endpoints ivolvaappsvc -n ivolve
```
```
NAME           ENDPOINTS                         AGE
ivolvaappsvc   10.244.1.8:3000,10.244.1.9:3000   95s
```

---

### 5) Verify Database Creation and Privileges in MySQL
Connect manually to MySQL in `ivolvedb-0` using root credentials from `ivolvesecret` to verify that `ivolve` database exists and `admin` user has the necessary privileges:
```bash
kubectl exec -it ivolvedb-0 -n ivolve -- mysql -u root -p123 -e "SHOW DATABASES; SHOW GRANTS FOR 'admin'@'%';"
```
```
+--------------------+
| Database           |
+--------------------+
| information_schema |
| ivolve             |
| mysql              |
| performance_schema |
| sys                |
+--------------------+
+---------------------------------------------------+
| Grants for admin@%                                |
+---------------------------------------------------+
| GRANT USAGE ON *.* TO `admin`@`%`                 |
| GRANT ALL PRIVILEGES ON `ivolve`.* TO `admin`@`%` |
+---------------------------------------------------+
```

---

### 6) Check Application Logs for MySQL Connectivity
Check the container logs of the `ivolveapp` pods to verify successful connection to the `ivolve` database:
```bash
kubectl logs -l app=ivolveapp -c app -n ivolve --tail=10
```
```
🔄 Attempting to connect to MySQL...
✅ Connected to MySQL and 'ivolve' DB found.
🚀 Server started on http://0.0.0.0:3000
🔄 Attempting to connect to MySQL...
✅ Connected to MySQL and 'ivolve' DB found.
🚀 Server started on http://0.0.0.0:3000
```

---

### 7) Test ClusterIP Service from Inside the Cluster
Execute HTTP requests to `http://ivolvaappsvc:3000` via the `network` diagnostic container in `ivolve` namespace:
```bash
kubectl exec -it network -n ivolve -- curl -s http://ivolvaappsvc:3000/health
```
```
🚀 iVolve web app is working! Keep calm and code on! 🎉
```
```bash
kubectl exec -it network -n ivolve -- curl -s http://ivolvaappsvc:3000/ready
```
```
👍 iVolve web app is ready to rock and roll! 🤘
```
```bash
kubectl exec -it network -n ivolve -- curl -s http://ivolvaappsvc:3000 | head -n 10
```
```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0"/>
  <title>iVolve | Global Cloud Company</title>
```
