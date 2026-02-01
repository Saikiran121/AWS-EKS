# Deep-Dive: MySQL Deployment & Internal Networking 🗄️🌐

Now that we have our **StorageClass**, **PVC**, and **ConfigMap** ready, it's time to assemble them into a working **MySQL Database**. This is where the magic of Kubernetes orchestration comes together.

---

## 1. The Architecture: Connecting the Dots 🧩

Our MySQL setup uses three distinct Kubernetes objects to ensure it is production-ready:
1.  **PVC (`ebs-mysql-pv-claim`)**: Ensures your data stays safe even if the pod is deleted.
2.  **ConfigMap (`usermanagement-dbcreation-script`)**: Automatically creates your database schema on the first boot.
3.  **Service (ClusterIP)**: Provides a stable DNS name (`mysql`) for your applications to connect to.

---

## 2. In-Depth: The Deployment Manifest 📄🐳

Below is the full content of `04-mysql-deployment.yaml`. Let's break down exactly what each part does.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mysql
  strategy:
    type: Recreate                  # 🔄 Re-mounts EBS volume by killing old pod first
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
              value: "dbpassword11" # 🔐 Password for the database
          ports:
            - containerPort: 3306   # 🚪 Standard MySQL port
              name: mysql
          volumeMounts:
            - name: mysql-persistent-storage
              mountPath: /var/lib/mysql        # 📂 Where actual DB data lives
            - name: usermanagement-dbcreation-script
              mountPath: /docker-entrypoint-initdb.d # 📜 Auto-runs SQL scripts
      volumes:
        - name: mysql-persistent-storage
          persistentVolumeClaim:
            claimName: ebs-mysql-pv-claim      # 🔗 Connects to the PVC we created
        - name: usermanagement-dbcreation-script
          configMap:
            name: usermanagement-dbcreation-script # 🔗 Connects to the ConfigMap
```

### Deep-Dive Analysis:
*   **`image: mysql:8.0`**: We use the official image which includes built-in logic for volume mounting and initialization scripts.
*   **`strategy: Recreate`**: Crucial for EBS! It ensures the volume is released from the old node before the new pod tries to grab it on a potentially different node.
*   **`volumeMounts` vs `volumes`**: 
    -   `volumes` defines **where** the data comes from (PVC/ConfigMap).
    -   `volumeMounts` defines **where** it appears inside the container's file system.
*   **`/docker-entrypoint-initdb.d`**: This is a "magical" directory. Any `.sql` file put here via a ConfigMap mount will be executed the very first time the database starts up.

---

## 3. In-Depth: The Service Manifest 📄🌐

Below is the full content of `05-mysql-clusterip-service.yaml`.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql                   # 📞 DNS name: 'mysql'
spec:
  selector:
    app: mysql                  # 🎯 Targets pods labeled 'app: mysql'
  ports:
    - port: 3306                # 🏢 Port used by internal applications
      targetPort: 3306          # 🏁 Port listening on the container
  type: ClusterIP               # 🏠 Internal Virtual IP (Standard for DBs)
```

### Deep-Dive Analysis:
*   **`type: ClusterIP`**: This is the default service type. It creates a stable internal IP that only other pods in the cluster can reach. It is **not** exposed to the internet.
*   **`metadata.name: mysql`**: This is extremely important. In Kubernetes, this name becomes a DNS entry. Your Java or Python apps don't need an IP address; they can just use `jdbc:mysql://mysql:3306/usermgmt`.
*   **`selector`**: This is how the Service knows which Pods to send traffic to. It must exactly match the labels in the Deployment's template.

---

## 4. How to Deploy 🚀

Run these commands in order:

```bash
# 1. Create the StorageClass and PVC
kubectl apply -f 01-storage-class.yaml
kubectl apply -f 02-persistent-volume-claim.yaml

# 2. Create the Configuration
kubectl apply -f 03-UserManagement-ConfigMap.yaml

# 3. Deploy MySQL and its Service
kubectl apply -f 04-mysql-deployment.yaml
kubectl apply -f 05-mysql-clusterip-service.yaml
```

---

## 5. Verification ✅

### Step 1: Check Pod Status
```bash
kubectl get pods -l app=mysql
```

### Step 2: Check PVC Binding
```bash
kubectl get pvc ebs-mysql-pv-claim
```
*It should now be in the `Bound` status.*

### Step 3: Test the Service
```bash
kubectl get svc mysql
```
*Note the 'CLUSTER-IP'. This is the stable internal IP.*

---

### Victory! 🏆
You have successfully deployed a stateful, auto-initializing database on AWS EKS! In the next section, we will deploy a **Java Spring Boot App** and connect it to this MySQL instance.
