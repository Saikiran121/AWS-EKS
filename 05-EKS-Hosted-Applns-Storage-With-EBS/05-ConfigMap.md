# Deep-Dive: Kubernetes ConfigMaps 💡⚙️

In a modern EKS environment, your application containers should be **immutable**. This means you should never bake environment-specific settings (like Database URLs or API endpoints) inside your Docker image. **ConfigMaps** are the tool that allow you to decouple your configuration from your code.

---

## 1. Why and When do we need a ConfigMap? 🤔

### 1.1 The Philosophy: Build Once, Run Anywhere 🌍
- **Why**: You want to use the **exact same Docker image** in Dev, Staging, and Production. 
- **The Problem**: Dev uses `dev-db.local`, while Prod uses `prod-db.aws.com`.
- **The Solution**: Your code looks for an environment variable named `DATABASE_URL`. You use ConfigMaps to inject the correct value based on the environment.

### 1.2 When to use it?
- Environment variables (e.g., Log Levels, Feature Flags).
- Configuration files (e.g., `nginx.conf`, `redis.conf`, `prometheus.yml`).
- Port numbers or common metadata.

> [!WARNING]
> **Do NOT use ConfigMaps for sensitive data!** ConfigMaps are stored in plain text. For passwords, API keys, or certificates, use **Kubernetes Secrets** instead.

---

## 2. Manifest Breakdown: The Data Store 🔍

A ConfigMap is essentially a dictionary of key-value pairs.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config               # 🏷️ Unique name in the namespace
  namespace: default
data:
  # Simple Key-Value
  LOG_LEVEL: "DEBUG"
  MAX_RETRIES: "5"
  
  # Multiline Configuration File
  database.properties: |
    db.host=mysql-prod.aws.com
    db.port=3306
    db.user=admin
```

---

## 3. Practical Example: Database Initialization Script 🗄️📜

A very common use case in EKS is to store SQL scripts in a ConfigMap to initialize a database when it first starts.

### The Manifest
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: usermanagement-dbcreation-script
data: 
  mysql_usermgmt.sql: |-
    DROP DATABASE IF EXISTS usermgmt;
    CREATE DATABASE usermgmt; 
```

### What does this do?
1.  **`mysql_usermgmt.sql`**: This is the "Key." When you mount this ConfigMap as a volume, Kubernetes will create a file named `mysql_usermgmt.sql` inside the pod.
2.  **`|-` (YAML Block Scalar)**: This special syntax is crucial for scripts. It allows you to write multiline text precisely:
    -   `|`: Preserves newlines.
    -   `-`: Strips the final newline at the end of the block (Chomping).
3.  **Automation**: MySQL and PostgreSQL Docker images have a special directory (`/docker-entrypoint-initdb.d`). If you mount this ConfigMap into that directory, the database will **automatically run this script** on startup.

---

## 4. How to Use ConfigMaps in Pods 🧩

There are two primary ways to consume a ConfigMap:

### 3.1 As Environment Variables (The Most Common)
Perfect for simple settings that your application reads at startup via `process.env` or `os.environ`.

```yaml
spec:
  containers:
    - name: my-app
      env:
        - name: APP_LOG_LEVEL
          valueFrom:
            configMapKeyRef:
              name: app-config   # 👈 Name of the ConfigMap
              key: LOG_LEVEL     # 👈 The specific key to read
```

### 3.2 As a Mounted Volume (For Config Files)
Perfect for software that expects to read settings from a physical file on disk (like Nginx or Redis).

```yaml
spec:
  containers:
    - name: nginx
      volumeMounts:
        - name: config-volume
          mountPath: /etc/nginx/conf.d  # 📂 Where files appear in the pod
  volumes:
    - name: config-volume
      configMap:
        name: app-config          # 👈 Mounts every key in 'data' as a file
```

---

## 5. Pro-Tips for Production 🚀

### 4.1 Live Updates (Hot Reloading) 🔄
If you update a ConfigMap that is mounted as a **Volume**, Kubernetes will automatically update the file inside the pod within a minute. 
*Note: ConfigMaps used as **Environment Variables** are static; the pod must be restarted to see the change.*

### 4.2 Size Limits 📏
A ConfigMap is stored in `etcd`. It has a strict limit of **1 MB**. If your configuration file is larger than that, it should probably be stored in S3 or EFS instead.

### 4.3 Immutable ConfigMaps 🛡️
In newer versions of Kubernetes, you can set `immutable: true`. This prevents accidental changes and improves cluster performance because the `kubelet` doesn't have to watch for updates.

---

## 6. Summary: ConfigMap vs Secret 🕵️‍♂️

| Feature | ConfigMap | Secret |
| :--- | :--- | :--- |
| **Data Type** | Non-sensitive (URLs, Flags) | Sensitive (Keys, Passwords) |
| **Storage** | Plaintext | Base64 Encoded (and encrypted at rest in EKS) |
| **Accessibility** | Visible in YAML/Logs | Obfuscated in YAML |

---

### Ready for the next level? 🚀
Configuration is useless without security. Let's learn how to handle sensitive data in **06-Kubernetes-Secrets.md**!
