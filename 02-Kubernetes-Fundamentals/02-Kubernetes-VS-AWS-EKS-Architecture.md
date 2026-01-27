# K8s Architecture vs. AWS EKS Architecture 🥊

This guide explains the difference between building your own Kubernetes cluster (Vanilla/Self-managed) and using the Amazon EKS managed service.

---

## 1. The Living Analogy 🏠
- **Vanilla Kubernetes:** Building a **Self-Built House**. You are responsible for the foundation, the wiring, the plumbing, and the security. If the roof leaks (Master Node dies), you have to fix it yourself at 3 AM.
- **Amazon EKS:** Staying in a **Managed Luxury Apartment**. AWS builds the foundation, manages the plumbing, and provides 24/7 security for the building (Control Plane). You only worry about your furniture and guests (Your Pods/Applications).

---

## 2. Who Manages What? (Shared Responsibility) 🛡️

### 2.1 The Control Plane (Master Nodes)
| Feature | Vanilla Kubernetes | Amazon EKS |
| :--- | :--- | :--- |
| **API Server & etcd** | You provision EC2 instances and install them. | AWS provisions and hides them from you. |
| **High Availability** | You must set up multiple masters and a Load Balancer. | AWS automatically runs them across 3 Availability Zones. |
| **Upgrades/Patching** | You run `kubeadm upgrade` and manage OS patches. | You click a button in the AWS Console. |
| **etcd Backups** | You manage snapshots and disaster recovery. | AWS automatically handles backups and recovery. |

### 2.2 The Worker Nodes
- **Vanilla K8s:** You must manually install the `kubelet`, `kube-proxy`, and join them to the cluster.
- **Amazon EKS:** AWS offers **Managed Node Groups**. You specify the instance type, and EKS handles the joining, scaling, and draining during updates.

---

## 3. The Cloud Integration Layer ☁️
One of the biggest differences is the **Cloud Controller Manager (CCM)**.

- **In Vanilla K8s:** You often have to install specific plugins to make Kubernetes talk to AWS (e.g., to create an ELB when you define a Service).
- **In Amazon EKS:** The integration is native. When you create a `Service` of type `LoadBalancer`, EKS immediately tells the AWS API to provision an NLB or ALB. It also integrates natively with **VPC Networking** via the **AWS VPC CNI**.

---

## 4. Why Use EKS? (The "DevOps Why") 🧠
For a 3YOE DevOps Engineer, the choice isn't just about "ease." It's about **SLA and Blast Radius**.

1.  **SLA (Service Level Agreement):** EKS provides a 99.95% availability SLA for the endpoint. In Vanilla K8s, *you* are the SLA.
2.  **AWS Integration:** EKS supports **IAM Roles for Service Accounts (IRSA)**, letting you give AWS permissions to a single Pod rather than the entire Node.
3.  **Cost vs. Effort:** While EKS costs $0.10/hour, the "human cost" of managing internal Master nodes, etcd clusters, and security patches usually far exceeds that dime.

---

## 💡 Quick Recap Table: Vanilla vs. EKS

| Component | Vanilla Kubernetes | AWS EKS |
| :--- | :--- | :--- |
| **Control Plane** | Self-Managed (Manual) | AWS Managed (Serverless) |
| **etcd** | Persistent Storage Setup (Manual) | Backend Managed by AWS |
| **Scaling Master** | Manual / Autoscaling Groups | Intelligent Auto-scaling by AWS |
| **Upgrades** | Risky / Manual | Controlled / Automated |
| **Networking** | Any CNI (Flannel, Calico) | Optimized **AWS VPC CNI** (Native VPC IPs) |
| **Responsibility** | Full Stack (Internal OS to K8s) | Upstream (Apps and Worker Nodes) |

> [!IMPORTANT]
> **Key Interview Takeaway:** In EKS, the Control Plane is "Serverless" from a user perspective. You can't SSH into the Master nodes, and you don't need to. This shifts your focus from **Cluster Administration** to **Application Reliability**.
