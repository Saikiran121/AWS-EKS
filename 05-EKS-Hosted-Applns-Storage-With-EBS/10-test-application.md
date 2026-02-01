# Hands-On: Deploy and Test UserManagement Service with MySQL Database 🧪🚀

This guide provides step-by-step instructions to deploy the User Management Microservice, connect it to a MySQL database, and verify its functionality.

---

## Step-01: Introduction 📜

We are going to deploy a **User Management Microservice** which connects to a **MySQL Database** schema named `usermgmt` during startup.

### APIs to Test:
*   **Create Users**: Add new users to the database.
*   **List Users**: Retrieve all created users.
*   **Delete User**: Remove a user by username.
*   **Health Status**: Verify the application's health.

### Kubernetes Objects:
| Object Type | YAML File |
| :--- | :--- |
| **Deployment & EnvVars** | `06-usermanagement-microservice-deployment.yaml` |
| **NodePort Service** | `07-usermanagement-microservice-nodeport-service.yaml` |

---

## Step-02: Deployment Manifests 📄

Ensure the following manifests are created in your `kube-manifests/` directory:
1.  **User Management Microservice Deployment**: Includes environment variables for DB connectivity (`DB_HOSTNAME`, `DB_PORT`, `DB_NAME`, `DB_USERNAME`, `DB_PASSWORD`).
2.  **NodePort Service**: Exposes the application on port `31231`.

---

## Step-03: Create UserManagement Service & Deployment 🚀

### 3.1 Apply Manifests
Run the following command to deploy the application and its service:
```bash
# Create Deployment & NodePort Service
kubectl apply -f kube-manifests/
```

### 3.2 Verify Deployment
```bash
# List Pods
kubectl get pods

# Verify logs of Usermgmt Microservice pod
kubectl logs -f <Pod-Name>

# Verify sc, pvc, pv
kubectl get sc,pvc,pv
```

### 🚨 Problem Observation:
If you deploy all manifests at once, the User Management Microservice pod might restart multiple times. 
- **Reason**: The application tries to connect to MySQL before the database is fully ready or initialized.
- **Solution**: We can use **initContainers** to wait for the database. We will explore this in a later section, but for now, Kubernetes' automatic restart policy will eventually allow the app to connect once MySQL is ready.

### 3.3 Access the Application
```bash
# List Services
kubectl get svc

# Get Worker Node Public IP
kubectl get nodes -o wide

# Access Health Status API
http://<EKS-WorkerNode-Public-IP>:31231/usermgmt/health-status
```

---

## Step-04: Troubleshooting Access (Missing AWS Configuration) ⚠️🛑

If your `curl` command or Postman request results in a **Connection Timeout**, the most likely cause is an AWS Security Group restriction.

### The Problem:
By default, EKS Worker Nodes are protected by a Security Group that does **not** allow external traffic on the NodePort range (`30000-32767`).

### The Fix:
You must manually add an inbound rule to your **Worker Node Security Group**:
1.  Open the **EC2 Console** -> **Instances**.
2.  Select one of your EKS Worker Nodes.
3.  Go to the **Security** tab and click on the **Security Group**.
4.  Click **Edit inbound rules**.
5.  Add a new rule:
    - **Type**: Custom TCP
    - **Port Range**: `30000-32767` 💡 (Open the **entire range** once so you don't have to do it for every service).
    - **Source**: `My IP` (Recommended for security) or `0.0.0.0/0`.
6.  Click **Save rules**.

---

## Step-05: Pro-Tip: How do Pros handle this? 🏎️🌬️

In a real production environment, you don't want to manually click around the AWS Console every time you add a service.

### 1. Opening the Entire Range
As mentioned above, opening the entire **30000-32767** range in your Security Group is a common "one-time" setup. This allows any `NodePort` service you create to work instantly without further SG changes.

### 2. AWS Load Balancer Controller (The Industry Standard)
The best way to handle this is to install the **AWS Load Balancer Controller (LBC)** in your cluster.
- **Automation**: When you create an `Ingress` or a `LoadBalancer` service, the LBC talks to AWS APIs and **automatically** updates your Security Groups for you.
- **Single Entry Point**: You use an **Ingress** to route traffic to dozens of microservices through a single AWS Application Load Balancer (ALB), so you only manage one entry point.

---

## Step-06: Test using Postman 📡

### 4.1 Setup
1.  Download [Postman](https://www.postman.com/downloads/).
2.  Import the collection: `AWS-EKS-Masterclass-Microservices.postman_collection.json`.
3.  **Create Environment**:
    - Name: `UMS-NodePort`
    - Variable: `url`
    - Value: `http://<WorkerNode-Public-IP>:31231`

### 4.2 Test APIs
Select the `UMS-NodePort` environment and call:
- **Health Status**: `{{url}}/usermgmt/health-status`
- **Create User**: `POST {{url}}/usermgmt/user`
  ```json
  {
      "username": "admin1",
      "email": "dkalyanreddy@gmail.com",
      "role": "ROLE_ADMIN",
      "enabled": true,
      "firstname": "fname1",
      "lastname": "lname1",
      "password": "Pass@123"
  }
  ```
- **List Users**: `GET {{url}}/usermgmt/users`
- **Delete User**: `DELETE {{url}}/usermgmt/user/admin1`

---

## Step-06: Verify Data in MySQL Database 🗄️

Connect to the database directly to verify the records:

```bash
# Connect to MYSQL Database
kubectl run -it --rm --image=mysql:8.0 --restart=Never mysql-client -- mysql -h mysql -u root -pdbpassword11

# Run SQL Commands
mysql> show schemas;
mysql> use usermgmt;
mysql> show tables;
mysql> select * from users;
```

---

## Step-07: Clean-Up 🧹

Delete all objects created in this section to save resources:

```bash
# Delete All
kubectl delete -f kube-manifests/

# Verify Cleanup
kubectl get pods
kubectl get sc,pvc,pv
```
