# Step-by-Step: Create AWS RDS Database 🗄️☁️

In this guide, we will provision a Managed MySQL database using AWS RDS. Our first priority is ensuring the database is placed in the **same VPC** as our EKS cluster to enable secure and low-latency communication.

---

## Step 1: Review EKS Cluster VPC 🗺️🛰️

Before creating the RDS instance, we must identify exactly where our EKS cluster is "living."

### 1.1 Identify VPC ID
1.  Log in to the **AWS Management Console**.
2.  Navigate to **Elastic Kubernetes Service (EKS)**.
3.  Click on your cluster name (e.g., `eksdemo1`).
4.  Go to the **Networking** tab.
5.  Under **Networking configuration**, locate the **VPC ID** (e.g., `vpc-0a1b2c3d...`). 
    *   **Action**: Copy this VPC ID. We will need it when creating the RDS instance.

### 1.2 Identify Subnets
1.  In the same **Networking** tab, look at the **Subnets** list.
2.  Note the **Availability Zones (AZs)** your cluster is using (e.g., `us-east-1a`, `us-east-1b`).
3.  **Pro Tip**: RDS requires at least **two subnets in different AZs** to create a DB Subnet Group. Make sure you know which private subnets your EKS nodes are using.

### 1.3 Review Security Groups
1.  Locate the **Additional security groups** or the **Cluster security group** link.
2.  Note the **Security Group ID** (e.g., `sg-0e5f6g7h...`).
3.  **Why?**: We will later configure the RDS Security Group to allow inbound traffic *only* from this specific EKS Security Group.

---

### Victory! 🏆
You have successfully mapped the network foundation of your EKS cluster. This ensures that when we create the RDS database in the next step, it will be perfectly aligned for connectivity.

---

## Step 2: Create DB Security Group 🛡️🔐

A Security Group acts as a virtual firewall for your RDS instance. We will create one that specifically allows traffic from our EKS nodes.

### 2.1 Create the Security Group
1.  Navigate to the **VPC Console** in the AWS Management Console.
2.  In the left sidebar, click on **Security Groups**.
3.  Click the **Create security group** button.
4.  **Security group name**: Enter `eks-rds-db-sg`.
5.  **Description**: Enter `Security group for RDS DB allowing access from EKS nodes`.
6.  **VPC**: Select the **VPC ID** you identified in Step 1 (e.g., `vpc-0a1b2c3d...`).

### 2.2 Configure Inbound Rules (The Most Important Part!)
1.  Under **Inbound rules**, click **Add rule**.
2.  **Type**: Select **MySQL/Aurora (3306)**.
3.  **Protocol**: Should be **TCP**.
4.  **Port range**: Should be **3306**.
5.  **Source**: 
    *   Change the dropdown to **Custom**.
    *   In the search box, paste the **Security Group ID** of your EKS cluster that you noted in Step 1 (e.g., `sg-0e5f6g7h...`).
    *   **Note**: This is called **Security Group Referencing**. It ensures that *only* pods running on your EKS worker nodes can talk to the database.
6.  Click **Create security group** at the bottom.

---

### Victory! 🏆
You have now built a secure gateway for your database. By referencing the EKS Security Group, you've ensured that no one from the outside world can touch your data, but your application pods have an open door.

---

## Step 3: Create DB Subnet Group 📂🏗️

RDS requires a **DB Subnet Group** to define which subnets it can use across different Availability Zones (AZs). This is mandatory for Multi-AZ deployments.

### 3.1 Navigate to RDS Subnet Groups
1.  Navigate to the **RDS Console** in the AWS Management Console.
2.  In the left sidebar, under **Configuration**, click on **Subnet groups**.
3.  Click the **Create DB subnet group** button.

### 3.2 Configure Subnet Group Details
1.  **Name**: Enter `eks-rds-subnet-group`.
2.  **Description**: Enter `Subnet group for EKS RDS database`.
3.  **VPC**: Select the same **VPC ID** you have been using (identified in Step 1).

### 3.3 Add Subnets (The Multi-AZ Part)
1.  **Availability Zones**: 
    *   Select at least **two** Availability Zones (e.g., `us-east-1a` and `us-east-1b`).
    *   **Pro Tip**: Choose the same AZs where your EKS worker nodes are running.
2.  **Subnets**: 
    *   Select the **Private Subnets** associated with those AZs.
    *   **Action**: Refer back to Step 1.2 where you identified your cluster's subnets.
3.  Click **Create** at the bottom.

---

### Victory! 🏆
You have now prepared the physical "residency" for your database. By creating a Subnet Group that spans two AZs, you've ensured that AWS RDS can provide high availability and failover for your data.

---

## Step 4: Create RDS Database (MySQL Engine) 🗄️⚡

Now that the networking foundation (VPC, SG, and Subnet Groups) is ready, we will provision the actual MySQL database instance.

### 4.1 Initiate Database Creation
1. Go to the **RDS Console**.
2. Click **Databases** in the left sidebar.
3. Click the **Create database** button.

### 4.2 Engine Options & Settings
1. **Choose a database creation method**: Select **Standard create**.
2. **Engine options**: Select **MySQL**.
3. **Engine Version**: Keep the default (e.g., `8.0.x`).
4. **Templates**: Select **Free Tier** (for testing) or **Dev/Test**.

### 4.3 Settings (Admin Credentials)
1. **DB instance identifier**: Enter `webappdb`.
2. **Master username**: Enter `dbadmin`.
3. **Master password**: Enter a secure password (e.g., `webappdb123`). 
    * **Action**: Note these credentials! We will use them in our Kubernetes Secret.

### 4.4 Instance Configuration & Storage
1. **Instance configuration**: Select `db.t3.micro` or `db.t3.small`.
2. **Storage**: keep defaults (20 GiB General Purpose SSD).

### 4.5 Connectivity (The Most Important Part!)
1. **Compute resource**: Select **Don't connect to an EC2 compute resource**.
2. **Network type**: IPv4.
3. **VPC**: Select the **VPC ID** you identified in Step 1.
4. **DB Subnet Group**: Select `eks-rds-subnet-group` (created in Step 3).
5. **Public access**: Select **No**.
6. **VPC security group**:
    * Choose **Existing**.
    * Remove the `default` security group.
    * Select `eks-rds-db-sg` (created in Step 2).
7. **Availability Zone**: Select "No preference" or a specific AZ from Step 1.

### 4.6 Additional Configuration
1. Expand **Additional configuration**.
2. **Initial database name**: Enter `usermgmt`.
3. Clear **Enable automated backups** if this is just a quick test (otherwise keep it enabled).
4. Click **Create database** at the bottom.

### 4.7 Create Kubernetes Secret for DB Password 🔐📂
Instead of putting our password directly in the deployment YAML, we store it in a **Kubernetes Secret**.

1. **Base64 Encode**: Kubernetes Opaque secrets require Base64 encoding.
   ```bash
   echo -n 'dbpassword11' | base64
   # Output: ZGJwYXNzd29yZDEx
   ```

2. **Create the Manifest**: Create `00-mysql-db-password-secret.yaml`.
   ```yaml
   apiVersion: v1
   kind: Secret
   metadata:
     name: mysql-db-password
   type: Opaque
   data: 
     db-password: ZGJwYXNzd29yZDEx
   ```

3. **Deploy the Secret**:
   ```bash
   kubectl apply -f 00-mysql-db-password-secret.yaml
   ```

---

### Victory! 🏆
Your managed database is now being provisioned! It will take a few minutes to move from `Creating` to `Available`. Once it is ready, note the **Endpoint** URL—this is the final piece of the puzzle to connect your EKS microservice to the cloud.

---

## Step 5: Create Kubernetes externalName service 🔗🛰️

Instead of hardcoding the long RDS endpoint in your application, we use a **Kubernetes Service of type ExternalName**. This maps an internal DNS name (e.g., `mysql`) to our external AWS RDS endpoint.

### 5.1 Why use ExternalName?
*   **Abstraction**: Your app only needs to know the name `mysql`.
*   **Portability**: If you move from a containerized MySQL to RDS, you don't change your app code—only your Service manifest.

### 5.2 Create the Service Manifest
Create a file named `01-mysql-externalname-service.yaml` in your workspace.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql
spec:
  type: ExternalName
  externalName: webappdb.xxxx.us-east-1.rds.amazonaws.com # 🛰️ Replace with your RDS Endpoint!
```

### 5.3 Deploy the Service
1.  **Get your Endpoint**: Go to the **RDS Console** -> **Databases** -> **webappdb** -> **Connectivity & security** -> **Endpoint**.
2.  **Update YAML**: Paste the endpoint into the `externalName` field of your manifest.
3.  **Apply to EKS**:
    ```bash
    kubectl apply -f 01-mysql-externalname-service.yaml
    ```
4.  **Verify**:
    ```bash
    kubectl get svc
    ```
    *   **The Result**: You should see a service named `mysql` with type `ExternalName` pointing to your RDS URL.

---

### Victory! 🏆
You have successfully bridged the gap between your Kubernetes cluster and the managed AWS RDS database. Your application can now resolve the name `mysql` to the powerful, managed database you just built.

---

## Step 6: Verify RDS Connectivity with Temporary Pod 🧪🧐

Before we deploy our full application, we must perform a **Sanity Check** to ensure that our networking (VPC, Security Groups, Subnet Groups) is actually working.

### 6.1 Why perform this test?
*   **Isolate Issues**: If the full app fails later, we know it's an app config issue, not a network issue.
*   **Validate Security Groups**: Confirms that your EKS nodes are allowed to talk to RDS on port 3306.
*   **Initialize Database**: Allows us to manually create the `usermgmt` database if it wasn't created during the RDS setup.

### 6.2 Step-by-Step Verification Commands

1.  **Check Service**: Confirm the `mysql` ExternalName service is active.
    ```bash
    kubectl get svc
    ```

2.  **Run Temporary Client**: Launch a one-off pod with a MySQL client and connect to your RDS endpoint.
    ```bash
    # Replace the hostname and password with your actual values
    kubectl run -it --rm --image=mysql:latest --restart=Never mysql-client -- mysql -h <YOUR-RDS-ENDPOINT> -u dbadmin -p<YOUR-PASSWORD>
    ```

3.  **Execute SQL Commands**: Inside the MySQL shell, verify your schemas and create the required database.
    ```sql
    -- Check existing databases
    show schemas;

    -- Create the application database
    create database usermgmt;

    -- Verify creation
    show schemas;

    -- Exit
    exit;
    ```

---

### Victory! 🏆
You have successfully connected to your cloud database from within your cluster! This proves that your EKS-to-RDS "Bridge" is strong and secure. Your infrastructure is now 100% ready for the production workload.

---

## Step 7: Update Application Deployment 🚀🚢

The final step is to deploy our microservice. We need to point it to the RDS instance and ensure it has the correct credentials.

### 7.1 What changed in the Manifest?
To transition from a local container DB to AWS RDS, we made the following critical updates in `02-usermgmt-microservice-with-rds.yaml`:

1.  **DB_HOSTNAME**: Changed from a local service name to `mysql`. Because of our **ExternalName** service (Step 5), this Resolve to the RDS endpoint.
2.  **DB_USERNAME**: Set to `dbadmin` (the master username we chose in Step 4.3).
3.  **DB_PASSWORD**: Uses `mysql-db-password` secret. **Important**: You must update your secret if you changed the master password during RDS creation.
4.  **InitContainer**: The `nc -z mysql 3306` command now checks if the RDS "Bridge" (ExternalName service) is responding before starting the application.

### 7.2 Deploy the Application
1.  **Update Secret (if needed)**:
    ```bash
    # Only run this if your master password changed
    kubectl create secret generic mysql-db-password --from-literal=db-password='<YOUR-RDS-PASSWORD>' --dry-run=client -o yaml | kubectl apply -f -
    ```

2.  **Apply Deployment**:
    ```bash
    kubectl apply -f 02-usermgmt-microservice-with-rds.yaml
    ```

3.  **Verify Rollout**:
    ```bash
    kubectl get pods
    kubectl logs -f deployment/usermgmt-microservice
    ```
    *   **Success Criteria**: The logs should show "Connected to MySQL database at mysql:3306".

---

### Congratulations! 🎖️🎉
You have successfully migrated your database layer from a simple container to a **Production-Grade Managed AWS RDS Instance**. Your EKS application is now backed by high availability, automated backups, and enterprise security.

---

## Step 8: Access the Application Externally (NodePort) 🌐📡

To access our microservice from outside the cluster (e.g., from your browser), we deploy a **NodePort Service**.

### 8.1 Create the Service Manifest
Create a file named `03-usermgmt-restapp-service.yaml` with the following content:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: usermgmt-restapp-service
  labels: 
    app: usermgmt-restapp
spec:
  type: NodePort
  selector:
    app: usermgmt-restapp
  ports: 
    - port: 8095
      targetPort: 8095
      nodePort: 31231
```

### 8.2 Deploy and Access
1.  **Apply the Service**:
    ```bash
    kubectl apply -f 03-usermgmt-restapp-service.yaml
    ```

2.  **Identify Node IP**:
    ```bash
    kubectl get nodes -o wide
    ```

3.  **Access URL**:
    Open your browser and navigate to:
    `http://<NODE-IP>:31231/usermgmt/health-status`

---

### Final Victory! 🏆🥇
You have completed the full production journey! Your microservice is running in EKS, securely connected to an AWS Managed RDS database, and safely accessible via a NodePort. 

---

## Step 9: Final Deployment and Database Verification 🚀✅

Now that all manifests are ready, we will perform a bulk deployment and verify that our application has successfully connected to RDS and initialized its schema.

### 9.1 Bulk Deploying All Resources
Instead of applying files one-by-one, we can apply the entire directory:

```bash
kubectl apply -f kube-manifests/
```
**Expected Output**:
* `secret/mysql-db-password created`
* `service/mysql configured`
* `deployment.apps/usermgmt-microservice created`
* `service/usermgmt-restapp-service created`

### 9.2 Monitor Pod Readiness
Watch the pod status until it reaches the `1/1 READY` state. This might take a minute as the `initContainer` waits for RDS and the Liveness/Readiness probes perform their checks.

```bash
kubectl get pods -w
```
*   **Why wait?**: The `1/1` state confirms that not only is the container running, but it has passed its health checks and is ready to serve traffic.

### 9.3 The "Moment of Truth": Database Schema Check
The User Management microservice is designed to automatically create its tables (JPA/Hibernate) when it connects to a fresh database. We will verify this from within our cluster.

1. **Connect to RDS**:
   ```bash
   kubectl run -it --rm --image=mysql:latest --restart=Never mysql-client -- mysql -h <YOUR-RDS-ENDPOINT> -u dbadmin -p<YOUR-PASSWORD>
   ```

2. **Verify Table Creation**:
   ```sql
   -- Switch to our app database
   use usermgmt;

   -- Check if Hibernate created the 'users' table
   show tables;
   ```
   **Expected Output**:
   ```text
   +--------------------+
   | Tables_in_usermgmt |
   +--------------------+
   | users              |
   +--------------------+
   ```

---

### Victory! 🏆🥇🗺️
If you see the `users` table, it means:
1.  **Network**: EKS successfully reached RDS (VPC/SG/Subnet Group worked).
2.  **DNS**: The `mysql` ExternalName service correctly resolved to your RDS endpoint.
3.  **Authentication**: The `mysql-db-password` secret was correctly consumed.
4.  **Application**: The microservice is fully healthy and stateful!

---

## 💡 Summary of the AWS RDS Flow
1.  **VPC Mapping**: Ensure EKS and RDS are in the same network.
2.  **Security**: Lock down the DB so only EKS pods can enter.
3.  **Availability**: Group subnets across zones.
4.  **Abstraction**: Use `ExternalName` to simplify the app configuration.
5.  **Connectivity**: Test first with a client pod.
6.  **Workload**: Deploy the app with RDS-specific environment variables.
7.  **Verification**: Confirm the application auto-created its database tables.
