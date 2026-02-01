# YAML Basics for Kubernetes: The "In-Depth" Guide 📄

YAML (YAML Ain't Markup Language) is a human-readable data serialization standard. In Kubernetes, it is used to define the **desired state** of your cluster objects (Pods, Deployments, Services, etc.).

---

## 1. Core Syntax & Rules 📏

YAML is extremely sensitive to formatting. If your indentation is off, Kubernetes will reject your file!

### 1.1 The Golden Rules
- **Indentation:** Use **Spaces**, NEVER tabs. Usually 2 spaces per level.
- **Case Sensitivity:** `Key` is different from `key`.
- **Key-Value Pairs:** Separated by a colon and a space (`key: value`). The **space after the colon** is mandatory!

---

## 2. Data Types: The Building Blocks 🧱

### 2.1 Scalars (Simple Values)
- **Strings:** `name: "my-app"` (Quotes are optional unless special characters exist).
- **Integers/Floats:** `replicas: 3` or `version: 1.2`.
- **Booleans:** `enabled: true` (or `false`).
- **Null:** `value: null` or `value: ~`.

### 2.2 Collections (Complex Values)

#### **Maps (Objects)**
Groups of key-value pairs. In K8s, `metadata` is a map.
```yaml
metadata:
  name: my-pod
  namespace: default
```

#### **Lists (Arrays)**
Items starting with a hyphen (`-`). Usually used for `containers` or `ports`.
```yaml
containers:
  - name: nginx
    image: nginx:latest
  - name: redis
    image: redis:alpine
```

---

## 3. Advanced Syntax: Multi-line Strings 📜

When you need to write a script or a long config inside YAML (e.g., in a ConfigMap).

- **Literal Block (`|`):** Preserves newlines and trailing spaces. Exactly what you type is what you get.
  ```yaml
  script: |
    echo "Hello"
    ls -l
  ```
- **Folded Block (`>`):** Replaces newlines with spaces. Good for long paragraphs.
  ```yaml
  description: >
    This is a long
    desc that will
    be one line.
  ```

---

## 4. Structure of a Kubernetes Manifest 🏗️

Every Kubernetes YAML file has 4 required top-level fields:

1.  **`apiVersion`**: Which version of the K8s API you are using (e.g., `v1`, `apps/v1`).
2.  **`kind`**: What type of object you are creating (e.g., `Pod`, `Service`, `Deployment`).
3.  **`metadata`**: Data that helps uniquely identify the object (e.g., `name`, `labels`, `annotations`).
4.  **`spec`**: The **Desired State**. This tells K8s what you want the object to look like (e.g., which image, how many replicas).

```yaml
# A Complete Example
apiVersion: v1
kind: Pod
metadata:
  name: example-pod
spec:
  containers:
    - name: nginx
      image: nginx:1.14.2
```

---

## 5. Multiple Documents in One File 📑

You can define a Service and a Deployment in the same `.yaml` file by separating them with three dashes (`---`).

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-svc
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-deploy
```

---

## 6. Pro Tips & Best Practices 💡

1.  **Use a Linter:** Use tools like `yamllint` or VS Code extensions to catch indentation errors.
2.  **Validate with Dry Run:** Before applying, run:
    `kubectl apply -f file.yaml --dry-run=client`
3.  **Comments:** Use `#` to document your manifests.
    ```yaml
    replicas: 3 # We need 3 for high availability
    ```
4.  **Avoid Tabs:** If you use Tabs, Kubernetes will throw a "mapping values are not allowed here" error.

---

## 7. Yamllint in Depth 🛡️

**Yamllint** is a command-line tool that checks YAML files for syntax errors and also for "cosmetic" issues like extra spaces, inconsistent indentation, or lines that are too long.

### 7.1 Why use Yamllint?
1.  **Catch "Invisible" Errors**: It detects tabs where spaces should be (the #1 cause of K8s YAML failures).
2.  **Enforce Team Standards**: Ensures every dev on your team uses the same indentation and line length.
3.  **CI/CD Integration**: Professionals use yamllint in GitHub Actions or Jenkins to stop "broken" YAML from ever reaching the cluster.

### 7.2 Installation
```bash
# Ubuntu/Debian
sudo apt-get install yamllint

# macOS
brew install yamllint

# Python/Pip (Cross-platform)
pip install yamllint
```

### 7.3 Basic Usage
```bash
# Check a single file
yamllint deployment.yaml

# Check all files in a directory
yamllint .
```

### 7.4 Understanding Output
- **`error`**: Fatal syntax issue. Kubernetes will definitely fail.
- **`warning`**: Style issue (e.g., "too many spaces"). Kubernetes might still accept it, but it's bad practice.

---

## 8. Quick Syntax Reference

| Feature | Syntax | Example |
| :--- | :--- | :--- |
| **Key-Value** | `key: value` | `app: nginx` |
| **List** | `- item` | `- containerPort: 80` |
| **Map** | `nested: key` | `metadata: name: my-app` |
| **Comment** | `# text` | `# Production config` |
| **Separator** | `---` | (Splits multiple objects) |
