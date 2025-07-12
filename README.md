# 🛠️ Lab: Kubernetes StatefulSet for MySQL with Persistent Storage

## 🎯 Objectives

- Deploy a stateful MySQL application using Kubernetes StatefulSet
- Ensure data persists through persistent volume claims (PVC)
- Use a headless service to enable stable DNS/hostname resolution
- Test network identity and persistence

---

## 📋 Prerequisites

- Kubernetes cluster (e.g., Minikube, Kind, EKS, etc.)
- `kubectl` configured
- Default StorageClass enabled (`minikube addons enable storage-provisioner` if using Minikube)

---

## 🚀 Task 1: Create the StatefulSet Manifest

### 📄 `mysql-statefulset.yaml`

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  serviceName: "mysql"
  replicas: 2
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql:8.0
        env:
        - name: MYSQL_ROOT_PASSWORD
          value: "password"
        ports:
        - containerPort: 3306
          name: mysql
        volumeMounts:
        - name: mysql-persistent-storage
          mountPath: /var/lib/mysql
  volumeClaimTemplates:
  - metadata:
      name: mysql-persistent-storage
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: "standard"
      resources:
        requests:
          storage: 1Gi
```

### ✅ Validate before applying

```bash
kubectl apply -f mysql-statefulset.yaml --dry-run=client
```

---

## ⚙️ Task 2: Deploy StatefulSet & Headless Service

### 📄 `mysql-service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql
spec:
  clusterIP: None
  ports:
  - port: 3306
    name: mysql
  selector:
    app: mysql
```

### 🧱 Apply both manifests

```bash
kubectl apply -f mysql-statefulset.yaml
kubectl apply -f mysql-service.yaml
```

### 🔍 Verify resources

```bash
kubectl get statefulset,pods,pvc
```

Expected:

```
NAME                     READY   AGE
statefulset.apps/mysql   2/2     1m

NAME            STATUS   VOLUME                                     CAPACITY
mysql-persistent-storage-mysql-0   Bound   pvc-xxxx   1Gi
mysql-persistent-storage-mysql-1   Bound   pvc-yyyy   1Gi
```

---

## 🧪 Task 3: Verify Data Persistence and DNS Resolution

### 🧼 Step 1: Create a test database

```bash
kubectl exec -it mysql-0 -- \
  mysql -uroot -ppassword -e "CREATE DATABASE lab_test;"
```

### 🗑️ Step 2: Delete Pod

```bash
kubectl delete pod mysql-0
```

Wait for a new `mysql-0` Pod to start, then:

```bash
kubectl exec -it mysql-0 -- \
  mysql -uroot -ppassword -e "SHOW DATABASES;"
```

✅ Expected: `lab_test` should still exist.

---

### 🌐 Step 3: DNS Resolution Test (requires `dig` or `nslookup`)

```bash
kubectl run -it --rm --image=busybox:1.28 test --restart=Never -- nslookup mysql
```

✅ Expected:
```
Name:      mysql
Address 1: 10.x.x.x mysql-0.mysql.default.svc.cluster.local
Address 2: 10.x.x.x mysql-1.mysql.default.svc.cluster.local
```

---

## 🧰 Troubleshooting Tips

| Problem                       | Fix                                                                 |
|------------------------------|----------------------------------------------------------------------|
| PVC stuck in Pending         | `kubectl get storageclass` → Make sure `standard` class exists       |
| Pod crash                    | `kubectl logs mysql-0` → Verify password, image, or volume mount     |
| DNS doesn’t resolve          | Ensure headless service is applied and Pods are running              |
| Minikube issue with volumes  | `minikube addons enable storage-provisioner`                         |

---

## 🧹 Cleanup

```bash
kubectl delete -f mysql-statefulset.yaml
kubectl delete -f mysql-service.yaml
```

---

## 👨‍💻 Author

**Abdullha Saleem**  
DevOps | Kubernetes | StatefulSets | Persistent Storage  
📫 *Let's connect on GitHub and LinkedIn for more real-world labs.*
