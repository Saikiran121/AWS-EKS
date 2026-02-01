# Hands-On: Testing the MySQL Connection 🧪🔌

After deploying your database, you need to verify that it's actually working. Since our MySQL service is a `ClusterIP`, we cannot access it from our local machine. We must test it from **inside** the EKS cluster.

---

## 1. The Strategy: The Temporary Client Pod 🏃‍♂️

Instead of creating a permanent deployment for testing, we will use a "one-off" pod that contains the MySQL client CLI. This pod will be automatically deleted after we exit the shell.

### Command breakdown:
```bash
kubectl run -it --rm --image=mysql:8.0 --restart=Never mysql-client -- mysql -h mysql -pdbpassword11
```

*   **`kubectl run mysql-client`**: Starts a new pod named `mysql-client`.
*   **`-it`**: Interactive terminal.
*   **`--rm`**: Automatically deletes the pod when you exit.
*   **`--restart=Never`**: 🛡️ Forces Kubernetes to create a "Pod" rather than a "Deployment" or "Job". This prevents the MySQL entrypoint from misinterpreting your command.
*   **`--image=mysql:8.0`**: Uses the official MySQL image which includes the `mysql` CLI tool.
*   **`--`**: Separates the `kubectl` arguments from the command we want to run inside the container.
*   **`mysql -h mysql -pdbpassword11`**:
    -   `-h mysql`: Connects to the host named `mysql` (this is the ClusterIP Service DNS name).
    -   `-p...`: The root password we defined in our deployment.

---

## 2. Step-by-Step Verification 🔍

Once you run the command above, you should see the `mysql>` prompt. Run the following SQL commands:

### 2.1 Verify Service Discovery
If the connection succeeds, it proves the **ClusterIP Service** is correctly routing traffic to the MySQL Pod.

### 2.2 Verify ConfigMap Initialization 📜
Remember the SQL script we put in the ConfigMap? Let's check if it worked:

```sql
# 1. List all databases
show databases;
```

**Expected Result:**
You should see `usermgmt` in the list. This proves that the MySQL entrypoint script correctly read and executed your ConfigMap volume mount!

```sql
# 2. Switch to the new database
use usermgmt;

# 3. Check for specific tables (if any were created)
show tables;

# 4. Exit the client
exit;
```

---

## 3. Advanced Test: Data Persistence 💾💎

If you want to prove the **EBS Storage** is working, try this experiment:

1.  Connect to MySQL again.
2.  Create a dummy table: `CREATE TABLE persist_test (id INT);`.
3.  Exit MySQL.
4.  **Kill the MySQL Pod**: `kubectl delete pod -l app=mysql`.
5.  Wait for the new Pod to reach `Running` status (recreated by the Deployment).
6.  Connect again using the `kubectl run` command from Section 1.
7.  Check the database: `use usermgmt; show tables;`.

**Result**: Your `persist_test` table will still be there! This proves the EBS volume was successfully detached from the old pod and re-attached to the new one without data loss.

---

## 4. Troubleshooting: "mysqld failed while attempting to check config" ⚠️🚫

If you run the test command and see an error like this:
`[ERROR] [Entrypoint]: mysqld failed while attempting to check config`
`[ERROR] [MY-013276] [Server] Failed to set datadir to '/usr/mysql/'`

### Why does this happen?
The official MySQL Docker image has a complex "Entrypoint" script. If you don't use the `--restart=Never` flag, `kubectl` might try to manage the pod in a way that causes the MySQL image to think it needs to start a **Server** process (`mysqld`) instead of just running the **Client** command (`mysql`).

Since the client pod doesn't have an EBS volume mounted to `/var/lib/mysql`, the server process fails immediately.

### The Fix:
Always include `--restart=Never` to ensure the pod is treated as a simple, one-time execution pod:
```bash
kubectl run -it --rm --image=mysql:8.0 --restart=Never mysql-client -- mysql -h mysql -pdbpassword11
```

---

## 5. General Troubleshooting Connection Issues 🛠️

If you get a "Connection Timed Out" or "Unknown Host" error:

1.  **Check Service Name**: Run `kubectl get svc`. Ensure the name is exactly `mysql`.
2.  **Check Pod Status**: Run `kubectl get pods`. The MySQL pod must be `Running`.
3.  **Check Labels**: Ensure the `selector` in the Service manifest exactly matches the `labels` in the Deployment template (`app: mysql`).
4.  **Check Logs**: If the pod is crashing, run `kubectl logs -l app=mysql`.

---

### Victory! 🏆
You have confirmed that your storage, configuration, and networking are all working perfectly together. You are now ready to connect your application!
