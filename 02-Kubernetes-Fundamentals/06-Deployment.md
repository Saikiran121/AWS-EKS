# Kubernetes Deployment: The "Masterclass" Guide (3YOE Context) 🏗️

A **Deployment** is the orchestrator for stateless applications in Kubernetes. It doesn't manage Pods directly; it manages **ReplicaSets**, which in turn manage **Pods**. This abstraction allows for seamless updates, version control, and effortless rollbacks.

---

## 1. Technical Architecture: The Chain of Command ⛓️

In an interview, you must explain the hierarchy. A Deployment provides declarative updates for Pods and ReplicaSets.

### The Relationship:
1.  **Deployment**: Manages the "Strategy" (RollingUpdate vs Recreate) and the "History" (Revisions).
2.  **ReplicaSet**: Manages the "Count" (Desired vs Current pods). It ensures $N$ pods are running at all times.
3.  **Pod**: The "Unit of Execution".

> [!NOTE]
> When you update a Deployment (e.g., change the image), the Deployment Controller creates a **new ReplicaSet** and gradually scales it up while scaling down the old one.

---

## 2. Verifying ReplicaSets: Why and How? 🕵️‍♂️

As a DevOps Engineer, you don't just check the Deployment status; you watch the **ReplicaSets**. 

### Why verify ReplicaSets?
1.  **Track Rollout Progress**: You can see exactly which ReplicaSet is gaining pods and which is losing them.
2.  **Debug Version Mismatches**: If you see 3 different ReplicaSets for one deployment, it means you've had 3 different "desired states" (revisions). Only one should eventually have pods.
3.  **Identify Stuck Updates**: If a new ReplicaSet is stuck at 0 replicas, but the deployment says "Progressing", the issue is likely at the RS level (e.g., node selector issues).

### How to verify:
```bash
# List all ReplicaSets
kubectl get rs

# Watch the transition (Best during an update)
kubectl get rs -w
```

---

## 3. Pod Verification: The "Last Mile" Check ✅

Verifying Pods is the most critical step. A Deployment can be "Successful" even if the application inside the pod is crashing!

### Why verify Pods after deployment?
1.  **Application Health**: A Pod can be `Running` but not `Ready` (Liveness/Readiness probes failing).
2.  **Runtime Errors**: Catch `CrashLoopBackOff`, `ImagePullBackOff`, or `OOMKilled` early.
3.  **Log Analysis**: Ensure that the application has started correctly and is connecting to its dependencies (DB, Cache).

### How to verify:
```bash
# 1. Broad Check: Status and Restarts
kubectl get pods -l app=my-web-app

# 2. Deep Dive: Why is it crashing?
kubectl describe pod <pod-name>

# 3. Application Brain: What is the app saying?
kubectl logs <pod-name> --tail=100
```

---

## 4. Rollout History: Why we NEED it? 📜

In a professional environment, checking the history is not just a "nice-to-have"; it is a **Risk Management** requirement.

### Why do we verify Rollout History?🦾
1.  **Auditing (Who & When)**: In a 3+ year DevOps role, you need to know who changed what and when. If the history shows `CHANGE-CAUSE`, you can trace a failure back to a specific `kubectl set image` or `apply` command.
2.  **Safety Before Rollback**: Moving blindly to "Revision 1" is dangerous. It might be a version from 6 months ago that has incompatible DB schemas! You verify history to identify the **last known stable** revision.
3.  **Troubleshooting Regressions**: When a bug appears on Tuesday, you check the history to see if any deployment happened on Monday night. This correlation is the fastest way to find the root cause.
4.  **Verification of Success**: After a CI/CD pipeline runs, you check history to ensure the revision incremented correctly.

### How to verify:
```bash
# 1. Broad view of all past lives
kubectl rollout history deployment/web-app

# 2. Deep investigation of a specific revision
# (Check this BEFORE you rollback!)
kubectl rollout history deployment/web-app --revision=5
```

> [!CAUTION]
> **The "Blind Rollback" Risk:** Never run `rollout undo` without checking history first. You might accidentally trigger a rollback to an ancient version that causes a permanent outage.

---

## 5. Problems Solved by a Deployment 🥊

| The Problem (Manual/ReplicaSet) | The Solution (With Deployment) |
| :--- | :--- |
| **Downtime Updates**: Manual pod deletion causes outages. | **Rolling Updates**: Zero-downtime transitions. |
| **The "Oops" Factor**: Re-applying old YAML is slow and risky. | **Native Rollbacks**: Instant `undo` to previous stable states. |
| **Update Tracking**: No built-in version history. | **Revision History**: Auditable trail of all changes. |

---

## 6. High-Level Capabilities 🛠️

1.  **Version Orchestration**: Moving from `v1` to `v2` safely using strategies.
2.  **Scaling**: Adjusting `replicas` to match traffic demand.
3.  **Self-Healing**: Automated replacement of failed pods via the ReplicaSet.
4.  **Rollback**: Returning to a known-good configuration.
5.  **Pause/Resume**: Critical for batching updates or performing "Canary-like" manual checks.

---

---

## 7. Scaling: The Professional Approach 📈

### 7.1 Imperative Scaling (Ad-hoc)
Best for sudden traffic spikes.
```bash
kubectl scale deployment/web-app --replicas=10
```

### 7.2 Declarative Scaling (Production)
Update the `yaml` file and apply.
```yaml
spec:
  replicas: 10
```
`kubectl apply -f deployment.yaml`

---

## 8. Update Methods: How to change Versions? 🔄

In Kubernetes, you have three primary ways to move your application from `v1` to `v2`.

### 8.1 Imperative (The Fast Way)
```bash
kubectl set image deployment/web-app nginx=nginx:1.21 --record
```

### 8.2 Declarative (The Professional/GitOps Way)
```bash
kubectl apply -f deployment.yaml
```

### 8.3 Interactive (The "Edit" Option) 📝
This opens the live configuration in your terminal's default editor.

**Step-by-Step:**
1.  Run: `kubectl edit deployment web-app`
2.  Change the `image:` version in the YAML.
3.  Save and exit. K8s triggers a **Rolling Update**.

---

## 9. Troubleshooting with `kubectl describe` 🔍

When a deployment fails, `kubectl describe` is your first line of defense. It reveals the **Events** and **Status** of the controllers.

### What to Look For:
```bash
kubectl describe deployment web-app
```

*   **Replicas**: Check `Desired`, `Updated`, `Total`, `Available`, and `Unavailable`.
*   **Conditions**: Look for `Progressing` and `Available` status.
*   **Events**: Look for `FailedCreate` or `ScalingReplicaSet` logs.

---

## 10. Advanced Operational Flow 🚀

### 10.1 The "Canary" Deployment Pattern 🐤
1.  **Update and Pause**:
    ```bash
    kubectl set image deployment/web-app nginx=nginx:v2.0
    kubectl rollout pause deployment/web-app
    ```
2.  **Verify**: Check logs and health of new pods.
3.  **Resume**: `kubectl rollout resume deployment/web-app`

### 10.2 Manual Batching (The Efficiency Trick) ⏸️
```bash
kubectl rollout pause deployment/web-app
kubectl set image deployment/web-app nginx=nginx:1.16.1
kubectl set resources deployment/web-app -c=nginx --limits=memory=512Mi
kubectl rollout resume deployment/web-app
```

---

## 11. Rollback Deep-Dive: The Time Machine ⏪

### Emergency Recovery
`kubectl rollout undo deployment/web-app`

### Auditing & Specific Rollbacks
1.  **Check History**: `kubectl rollout history deployment/web-app`
2.  **Rollback to specific version**:
    ```bash
    kubectl rollout undo deployment/web-app --to-revision=3
    ```

> [!TIP]
> Use `revisionHistoryLimit` in your YAML (e.g., `revisionHistoryLimit: 10`) to control how many old ReplicaSets are retained for rollbacks.

---

## 12. 3YOE Interview Scenarios (Q&A) 🎤

### Q1: "A Deployment update is stuck. How do you investigate?"
- **Answer**: I use a "Check-Down" methodology:
  1.  **Deployment**: `kubectl rollout status` and `kubectl describe` for event logs.
  2.  **ReplicaSet**: `kubectl get rs` to see if the new RS is scaling up.
  3.  **Pods**: `kubectl get pods` for statuses like `ImagePullBackOff` and `kubectl logs` for app errors.

### Q2: "Why is it important to check Rollout History before a rollback?"
- **Answer**: Because revision numbers are incremental. Revision 1 might be very old. I verify the history to find the *last known stable* revision (e.g., Revision 5) to ensure I don't roll back to a version with different dependencies or outdated configurations.

### Q3: "What is the difference between `maxSurge` and `maxUnavailable`?"
- **Answer**: `maxSurge` is extra capacity (e.g., +1 pod), while `maxUnavailable` is how many can be down during the transition (e.g., -1 pod).

---

## 💡 Quick Recall Table

| Feature | CLI Command | Professional Utility |
| :--- | :--- | :--- |
| **Edit** | `kubectl edit` | Interactive version/config tweaks |
| **Rollback** | `kubectl rollout undo` | Rapid failure recovery |
| **History** | `kubectl rollout history` | Auditing and version tracking |
| **Pause** | `kubectl rollout pause` | Canary testing & Batch updates |
| **Status** | `kubectl rollout status` | CI/CD pipeline health checks |
| **Inspect** | `kubectl describe` | Deep troubleshooting & Event logs |
| **Verify RS** | `kubectl get rs` | Rollout tracking & Debugging |
| **Verify Pods** | `kubectl get pods` | Application health & Runtime logs |
| **Audit** | `kubectl rollout history` | Version correlation & Risk check |
