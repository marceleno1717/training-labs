# Lab 13: Persistent Storage Setup for Application Logging

### 1) Create PersistentVolume Manifest (pv.yaml)
Define a cluster-scoped PersistentVolume (`ivolvepv`) with `1Gi` capacity, `hostPath` storage at `/mnt/app-logs`, `ReadWriteMany` access mode, and `Retain` reclaim policy:
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: ivolvepv
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: /mnt/app-logs
```

Apply the PersistentVolume:
```bash
kubectl apply -f pv.yaml
```

### 2) Create PersistentVolumeClaim Manifest (pvc.yaml)
Define a PersistentVolumeClaim (`ivolvepvc`) in the `ivolve` namespace requesting `1Gi`, with `ReadWriteMany` access mode, without using any StorageClass (`storageClassName: ""`):
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: ivolvepvc
  namespace: ivolve
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
  storageClassName: ""
```

Apply the PersistentVolumeClaim:
```bash
kubectl apply -f pvc.yaml
```

### 3) Verify PersistentVolume and Claim Status
Check the PersistentVolume status (should transition to `Bound`):
```bash
kubectl get pv ivolvepv
```
```
NAME       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM               STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
ivolvepv   1Gi        RWX            Retain           Bound    ivolve/ivolvepvc                   <unset>                 1m
```

Check the PersistentVolumeClaim status in the `ivolve` namespace:
```bash
kubectl get pvc ivolvepvc -n ivolve
```
```
NAME        STATUS   VOLUME     CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
ivolvepvc   Bound    ivolvepv   1Gi        RWX                           <unset>                 1m
```

### 4) Verify Binding Details
Inspect the PVC to verify volume binding and specifications:
```bash
kubectl describe pvc ivolvepvc -n ivolve
```
```
Name:          ivolvepvc
Namespace:     ivolve
StorageClass:  
Status:        Bound
Volume:        ivolvepv
Labels:        <none>
Annotations:   pv.kubernetes.io/bind-completed: yes
               pv.kubernetes.io/bound-by-controller: yes
Capacity:      1Gi
Access Modes:  RWX
VolumeMode:    Filesystem
Used By:       <none>
```

---

### 5) Test Application Pod Consuming Storage and Secrets
*(Optional test pod to verify mounting the PVC alongside the secret created in Lab 12):*
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: storage-test-pod
  namespace: ivolve
spec:
  containers:
    - name: test-container
      image: busybox
      command: ["sh", "-c", "echo 'Log test' >> /mnt/logs/app.log && sleep 3600"]
      env:
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: ivolvesecret
              key: DB_PASSWORD
      volumeMounts:
        - name: log-volume
          mountPath: /mnt/logs
  volumes:
    - name: log-volume
      persistentVolumeClaim:
        claimName: ivolvepvc
```
