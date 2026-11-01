# Lab 16: Pod Resource Management with CPU and Memory Requests and Limits

### 1) Install Metrics Server Using Helm
Add the official metrics-server Helm repository and update the chart index:
```bash
helm repo add metrics-server https://kubernetes-sigs.github.io/metrics-server/ && helm repo update
```

Install the `metrics-server` release in the `kube-system` namespace (enabling insecure TLS for local development/Kubelet communication):
```bash
helm install metrics-server metrics-server/metrics-server -n kube-system --set "args={--kubelet-insecure-tls,--kubelet-preferred-address-types=InternalIP\,ExternalIP\,Hostname}"
```

Verify that the metrics-server deployment is ready:
```bash
kubectl rollout status deployment/metrics-server -n kube-system
```
```
deployment "metrics-server" successfully rolled out
```

---

### 2) Update Deployment Manifest with Resource Requests and Limits (deployment.yaml)
Update `deployment.yaml` for the `ivolveapp` Deployment in the `ivolve` namespace to specify resource requests and limits for the application container:
- **Requests**:
  - `cpu`: `1` (1 vCPU)
  - `memory`: `1Gi`
- **Limits**:
  - `cpu`: `2` (2 vCPUs)
  - `memory`: `2Gi`

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
          resources:
            requests:
              cpu: "1"
              memory: "1Gi"
            limits:
              cpu: "2"
              memory: "2Gi"
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

Apply the updated Deployment:
```bash
kubectl apply -f deployment.yaml
```

Wait for the rollout to complete:
```bash
kubectl rollout status deployment/ivolveapp -n ivolve
```
```
deployment "ivolveapp" successfully rolled out
```

---

### 3) Verify Applied Resource Requests and Limits Using kubectl describe pod
Inspect the updated pod specification to verify the applied resource constraints:
```bash
kubectl describe pod -l app=ivolveapp -n ivolve | grep -A 8 "Limits:"
```
```
    Limits:
      cpu:     2
      memory:  2Gi
    Requests:
      cpu:     1
      memory:  1Gi
```

Or inspect a specific pod directly:
```bash
kubectl describe pod $(kubectl get pods -n ivolve -l app=ivolveapp -o jsonpath="{.items[0].metadata.name}") -n ivolve | grep -A 8 "Limits:"
```
```
    Limits:
      cpu:     2
      memory:  2Gi
    Requests:
      cpu:     1
      memory:  1Gi
```

---

### 4) Monitor Real-Time Resource Usage Using kubectl top pod
Query real-time CPU and memory metrics for the running application pods:
```bash
kubectl top pod -n ivolve -l app=ivolveapp
```
```
NAME                        CPU(cores)   MEMORY(bytes)   
ivolveapp-6d5db6865-ldt8p   1m           20Mi            
ivolveapp-6d5db6865-skwlb   1m           107Mi           
```

Monitor all pods in the `ivolve` namespace:
```bash
kubectl top pod -n ivolve
```
```
NAME                        CPU(cores)   MEMORY(bytes)   
ivolveapp-6d5db6865-ldt8p   1m           20Mi            
ivolveapp-6d5db6865-skwlb   1m           107Mi           
ivolvedb-0                  15m          507Mi           
network                     0m           3Mi             
```
