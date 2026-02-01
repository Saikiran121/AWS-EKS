# Deep-Dive: User Management Microservice Deployment ☕🚀

Now that our MySQL database is running and accessible within the cluster, it's time to deploy the **Java Spring Boot User Management Microservice**. This application will "plug into" the database using the internal networking and credentials we've configured.

---

## 1. The Deployment Manifest 📄🐳

Below is the full content of `06-usermanagement-microservice-deployment.yaml`. This manifest defines how our application container should be run in EKS.

```yaml
apiVersion: apps/v1
kind: Deployment 
metadata:
  name: usermgmt-microservice
  labels:
    app: usermgmt-restapp
spec:
  replicas: 1
  selector:
    matchLabels:
      app: usermgmt-restapp
  template:  
    metadata:
      labels: 
        app: usermgmt-restapp
    spec:
      containers:
        - name: usermgmt-restapp
          image: stacksimplify/kube-usermanagement-microservice:1.0.0
          ports: 
            - containerPort: 8095           # 🚪 Application port (Spring Boot default)
          env:
            - name: DB_HOSTNAME
              value: "mysql"                # 📞 Internal DNS name of the MySQL Service
            - name: DB_PORT
              value: "3306"            
            - name: DB_NAME
              value: "usermgmt"            # 🗄️ Database created by our ConfigMap
            - name: DB_USERNAME
              value: "root"            
            - name: DB_PASSWORD
              value: "dbpassword11"         # 🔐 Must match MySQL_ROOT_PASSWORD
```

---

## 2. Exposing the App: The NodePort Service 📄🌐

While the Deployment runs our application, it is still isolated inside the cluster. To access it from the outside world (or via a Load Balancer later), we need a **Service**. Below is the full content of `07-usermanagement-microservice-nodeport-service.yaml`.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: usermgmt-restapp-service
  labels: 
    app: usermgmt-restapp
spec:
  type: NodePort                  # 🚪 Exposes the service on a specific port on each Node
  selector:
    app: usermgmt-restapp         # 🎯 Targets the pods from our Deployment
  ports: 
    - port: 8095                  # 🏛️ The Port visible inside the cluster
      targetPort: 8095            # 🏁 The Port listening on the container
      nodePort: 31231             # 📍 The Port opened on every EKS Worker Node IP
```

### Deep-Dive Analysis:
*   **`type: NodePort`**: This service type opens a specific port on **every worker node** in your EKS cluster. Traffic sent to `<Node-IP>:31231` will be forwarded to your application.
*   **The Three Ports Explained**:
    1.  **`port (8095)`**: The port used by other services *inside* the cluster to reach this app.
    2.  **`targetPort (8095)`**: The port your Java application is actually listening on.
    3.  **`nodePort (31231)`**: The external port (must be in the 30000-32767 range) that makes the app accessible via the Node's IP address.
*   **`selector`**: Just like the MySQL service, this must match the `labels` in your Deployment to ensure the "pipe" is connected to the right pods.

---

## 3. In-Depth Configuration Explanation 🕵️‍♂️

### 2.1 Service Discovery via DNS (`DB_HOSTNAME`)
Notice the `DB_HOSTNAME` value is simply `"mysql"`. 
- **How it works**: Kubernetes has a built-in DNS service (`CoreDNS`). When we created the `mysql` Service in Section 06, it registered the name `mysql` in the cluster's internal DNS.
- **Why it matters**: The Java application doesn't need to know the dynamic IP of the MySQL pod. It just "calls" the name `mysql`, and Kubernetes routes the request to the correct pod.

### 2.2 Reaching the correct Database (`DB_NAME`)
The `DB_NAME` is set to `"usermgmt"`.
- This matches the database name we specified in our `03-UserManagement-ConfigMap.yaml` script.
- Since we mounted that script into the MySQL `/docker-entrypoint-initdb.d` directory, the database exists and is ready for the Spring Boot app to connect to.

### 2.3 Port Mapping (`containerPort`)
The container is listening on port `8095`. This is specific to this microservice's internal configuration. We must ensure that any future `Service` we create for this app targets this exact port.

---

## 4. Deployment Steps 🚀

1.  **Ensure MySQL is Running**:
    ```bash
    kubectl get pods -l app=mysql
    ```

2.  **Apply the Deployment**:
    ```bash
    kubectl apply -f 06-usermanagement-microservice-deployment.yaml
    ```

3.  **Check Status**:
    ```bash
    kubectl get pods -l app=usermgmt-restapp
    ```

---

## 5. Summary: The Connector Pattern 🔌

This manifest is a perfect example of the **Connector Pattern** in Kubernetes:
- **Portability**: The Docker image is generic (`stacksimplify/kube-usermanagement-microservice`).
- **Configuration**: All "environment-specific" details (DB host, credentials) are injected via the `env` block.
- **Discovery**: We rely on internal DNS rather than fragile IP addresses.

---

### What's Next? 🌐
Currently, our microservice is running but it's not accessible from the outside world. In the next section, we will create a **NodePort Service** to expose this application to users!
