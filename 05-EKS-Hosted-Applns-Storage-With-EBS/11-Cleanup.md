# Comprehensive Cleanup: Decommissioning your EKS Environment 🧹🧨

To avoid unexpected AWS costs and ensure your environment is clean, follow this systematic cleanup guide. We will work from the "Inside-Out"—starting with the application and ending with the cluster itself.

---

## Phase 1: Kubernetes Resource Cleanup 🏗️🗑️

First, remove all the objects we created inside the cluster.

### 1.1 Delete Applications and Services
```bash
# Delete all manifests in the kube-manifests folder
kubectl delete -f kube-manifests/
```

### 1.2 Delete Persistent Components
Wait for the apps to delete, then remove the storage claims.
```bash
# Delete PVCs (This will trigger the deletion of the EBS volumes)
kubectl delete pvc --all

# Delete the StorageClass
kubectl delete -f 01-storage-class.yaml
```

### 1.3 Verify K8s Cleanup
```bash
kubectl get all
kubectl get pvc,pv,sc
```
*Ensure no 'Bound' volumes or 'Running' pods remain.*

---

## Phase 2: AWS EKS Add-on & Identity Cleanup 🆔🛡️

If you installed the EBS CSI driver via Pod Identity, you must clean up those associations.

### 2.1 Delete Pod Identity Association
```bash
aws eks delete-pod-identity-association \
  --cluster-name eksdemo1 \
  --association-id <YOUR-ASSOCIATION-ID>
```
*(You can find the ID using `aws eks list-pod-identity-associations --cluster-name eksdemo1`)*

### 2.2 Remove EKS Add-ons
```bash
aws eks delete-addon --cluster-name eksdemo1 --addon-name aws-ebs-csi-driver
```

---

## Phase 3: IAM Cleanup (The "Security" Sweep) 🔒🧤

Remove the roles and policies that allowed our pods to talk to AWS services.

### 3.1 Delete IAM Role and Policy
1.  Go to the **IAM Console** -> **Roles**.
2.  Search for your EBS CSI role (e.g., `eks-ebs-csi-driver-role`).
3.  Delete the role (this will also detach the policy).

---

## Phase 4: Full Cluster Deletion 💥🪐

Once the dependencies are gone, delete the worker nodes and the control plane.

### 4.1 Using eksctl (Recommended)
If you created your cluster with `eksctl`, this command handles everything (VPC, Nodes, Control Plane):
```bash
eksctl delete cluster --name eksdemo1
```

### 4.2 Using AWS Management Console
1.  Delete the **Node Groups** first (EKS Console -> Cluster -> Compute).
2.  Once Node Groups are gone, delete the **EKS Cluster**.
3.  Manually delete the **VPC** if it was created specifically for this cluster.

---

## Phase 5: Final Cost Audit (The "Wallet" Save) 💰👀

Sometimes AWS resources can "leak" even after cluster deletion. Check the following consoles:

1.  **EC2 Console -> Volumes**: Ensure no EBS volumes from our PVCs are still "Available." If they are, delete them.
2.  **EC2 Console -> Elastic IPs**: Ensure no Load Balancer EIPs are still allocated.
3.  **CloudWatch Logs**: Delete the log groups created by the EKS cluster (under `/aws/eks/...`).

---

### Victory! 🏆
You have successfully decommissioned your entire EKS storage environment. You've learned how to build it, how to scale it, and how to safely tear it down!
