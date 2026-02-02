# Deep-Dive: Resource Quotas 📊⚖️💰

If **LimitRanges** are the "Guardrails" for individual containers, **Resource Quotas** are the "Total Budget" for an entire Namespace. They ensure that one team or environment cannot consume all the resources of a physical EKS cluster.

---

## 1. What is a ResourceQuota? (The Total Budget) 🏛️

A **ResourceQuota** provides constraints that limit the **total** resource consumption per namespace. 

### Analogy: The Shared Credit Card 💳
- **LimitRange**: "You cannot buy any single item more expensive than $100."
- **ResourceQuota**: "The entire family cannot spend more than $1,000 this month across all items."

In Kubernetes:
- **LimitRange**: "No single Pod can exceed 512Mi of RAM."
- **ResourceQuota**: "The total RAM of all Pods in the `dev` namespace cannot exceed 10Gi."

---

## 2. Types of Quotas (The "What") 🔍

Kubernetes can limit three categories of resources:

### 2.1 Compute Resources
Limits the sum of all requests and limits in the namespace.
- `requests.cpu`, `requests.memory`
- `limits.cpu`, `limits.memory`

### 2.2 Storage Resources
Limits the total storage used by PVCs.
- `requests.storage`
- `persistentvolumeclaims` (total number)

### 2.3 Object Counts
Limits the number of specific objects.
- `pods`, `services`, `secrets`, `configmaps`

---

## 3. Real-World Implementation 📄🛠️

In [18-dev-resourcequota.yaml](file:///home/user/Skills/EKS/06-Secrets-InitContainers-LivenessProbes-Namespace-RequestLimits/kube-manifests/18-dev-resourcequota.yaml), we set a strict budget for our `dev` namespace:

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: dev-quota
  namespace: dev
spec:
  hard:
    # Compute Budget
    requests.cpu: "1"        # Total 1 vCPU requested
    requests.memory: 1Gi     # Total 1Gi RAM requested
    limits.cpu: "2"          # Total 2 vCPU limit
    limits.memory: 2Gi       # Total 2Gi RAM limit
    # Object Budget
    pods: "10"               # Max 10 pods
    services: "5"            # Max 5 services
```

---

## 4. Why use them? (The "Why") ❓

1.  **Multi-Tenancy Cost Control**: Prevent Team A from deploying 100 replicas and blowing your AWS bill.
2.  **Cluster Stability**: Ensure the EKS Control Plane and system components (CoreDNS, CNI) always have enough "unbudgeted" head-room to operate.
3.  **Fair Sharing**: Distribute a fixed-size cluster (e.g., 3 `m5.large` nodes) fairly across multiple teams.

---

## 5. Violation Behavior 🛑💥

If a new Pod would push the namespace consumption above the quota:
- **API Rejection**: The Kubernetes API server rejects the request.
- **Error**: `Error from server (Forbidden): pods "xxx" is forbidden: exceeded quota: dev-quota, requested: requests.memory=256Mi, used: requests.memory=900Mi, limited: requests.memory=1Gi`.

---

## 6. Monitoring Quotas 🧪👀

How do you know how much budget is left?
```bash
# Describe the quota in your namespace
kubectl describe quota dev-quota -n dev
```
**The output will show**:
- `Resource`: The thing being limited.
- `Used`: How much is currently consumed.
- `Hard`: The max limit.

---

### Victory! 🏆
You have now implemented **Resource Budgeting**. By combining LimitRanges and ResourceQuotas, you have complete control over your EKS cluster's resources, enabling a safe, multi-tenant environment for your entire organization. 📊⚖️🚀
