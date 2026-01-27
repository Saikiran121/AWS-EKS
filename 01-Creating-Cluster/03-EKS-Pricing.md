# AWS EKS Pricing Guide: Professional Breakdown 💰

As a DevOps Engineer, understanding the cost structure is as important as the architecture. Here is the breakdown of EKS pricing for a 3YOE professional context.

---

## 1. The Cluster Cost (Control Plane)
**Price:** **$0.10 per hour** per cluster.
- **Monthly Cost:** Approx. **$72/month**.
- **What it covers:** AWS manages the Kubernetes Control Plane (API Server, etcd) across multiple Availability Zones for high availability.
- **Interview Detail:** You pay this regardless of the cluster size or node count. Scaling the control plane is managed by AWS at no extra cost.

## 2. Worker Node Costs (Compute)
Worker nodes are where your applications actually run. You pay for what you use.

| Compute Type | Pricing Model | Best Use Case |
| :--- | :--- | :--- |
| **EC2 (Managed Node Groups)** | Standard EC2 Instance Pricing | Legacy apps, steady workloads, custom OS needs. |
| **Fargate (Serverless)** | Pay per vCPU and GB of memory per second | Microservices, batch jobs, "set it and forget it" compute. |
| **Spot Instances** | Up to 90% off EC2 prices | Fault-tolerant apps, stateless microservices, CI/CD runners. |

- **Interview Tip:** Mention that **Managed Node Groups** use standard EC2 pricing, but **EKS Fargate** has its own pricing (approx. $0.04 per vCPU-hour and $0.004 per GB-hour).

## 3. Support Tiers: Standard vs Extended
Kubernetes versions move fast. AWS offers two support modes:
- **Standard Support:** **$0.10/hour** (base price). Applies to the latest versions.
- **Extended Support:** **$0.60/hour total** ($0.10 + $0.50 surcharge).
- **Why?** If you stay on an old version of Kubernetes (e.g., 1.25 when 1.30 is out), AWS charges a premium to keep that control plane secure and supported.
- **DevOps Strategy:** Always aim for "N-1" or "N-2" versioning to avoid the extended support surcharge.

## 4. Associated (Hidden) Costs
The cluster itself is only part of the bill. Don't forget:
- **Load Balancers (ALB/NLB):** Approx. $16/month + processed data fees.
- **EBS Storage:** Fees for the GP3/GP2 volumes attached to your nodes.
- **Data Transfer:** Inter-AZ traffic ($0.01/GB) and Internet Egress.
- **VPC Endpoints:** If you access AWS services privately, each endpoint adds cost.

## 5. Cost Optimization Strategies (The 3YOE Value-Add)
In an interview, mention these to show you care about the company's wallet:
1.  **Spot Instances:** Use `eksctl` or Karpenter to leverage Spot for stateless workloads.
2.  **Karpenter:** A high-performance auto-scaler that picks the cheapest instance type for your specific pod needs.
3.  **Horizontal Pod Autoscaler (HPA):** Scale pods down at night to trigger the Cluster Autoscaler to terminate unneeded EC2 nodes.
4.  **Savings Plans:** Commit to a certain amount of compute for 1-3 years to get ~30-50% discounts.

---

## 💡 Quick Recall Table: EKS Pricing

| Item | Cost Hook | "At a Glance" |
| :--- | :--- | :--- |
| **Cluster** | "Dime an hour" | $72/month |
| **Old Version** | "Six dimes an hour" | $432/month (Avoid this!) |
| **Nodes** | "Standard EC2" | Varies by instance size |
| **Spot** | "90% off" | Massive savings for stateless |
| **Inter-AZ** | "The Hidden Penny" | $0.01/GB (Watch out for this) |
