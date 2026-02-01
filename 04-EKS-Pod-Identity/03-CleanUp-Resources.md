# Clean-Up: Safely Deleting EKS Pod Identity Resources 🧹

Deleting resources in the wrong order can result in "orphaned" IAM roles or stuck clusters. Follow this **Reverse Order** guide to clean up your environment completely.

---

## 1. The Golden Rule: Deletion Order ⚖️

1.  **Pod Identity Association**: Remove the link first.
2.  **Pod Identity Agent Add-on**: Remove the agent from nodes.
3.  **Kubernetes Resources**: Delete the Pods and ServiceAccounts.
4.  **IAM Role**: Delete the AWS Identity.
5.  **EKS Cluster**: Delete the cluster itself.

---

## 2. Method A: AWS CLI (Automated) 💻

Run these commands in order:

### Step 1: Delete Pod Identity Association
Find the ID and delete it:
```bash
# List associations
aws eks list-pod-identity-associations --cluster-name $CLUSTER_NAME

# Delete specific association
aws eks delete-pod-identity-association \
  --cluster-name $CLUSTER_NAME \
  --association-id <ASSOCIATION_ID>
```

### Step 2: Delete the Add-on
```bash
aws eks delete-addon \
  --cluster-name $CLUSTER_NAME \
  --addon-name eks-pod-identity-agent
```

### Step 3: Delete Kubernetes Manifests
```bash
kubectl delete -f aws-cli-pod.yaml
kubectl delete -f my-service-account.yaml
```

### Step 4: Delete the IAM Role
```bash
# Detach the policy first
aws iam detach-role-policy \
  --role-name eks-pod-identity-s3-role \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess

# Delete the role
aws iam delete-role --role-name eks-pod-identity-s3-role
```

### Step 5: Delete the Cluster
```bash
aws eks delete-cluster --name $CLUSTER_NAME
```

---

## 3. Method B: AWS Management Console (Visual) 🖥️

### Step 1: Pod Identity Association
1.  Go to **EKS Console** -> Select your **Cluster**.
2.  Click the **Access** tab.
3.  Scroll to **Pod Identity associations**.
4.  Select the association and click **Delete**.

### Step 2: Add-on
1.  Click the **Add-ons** tab in EKS.
2.  Select **EKS Pod Identity Agent**.
3.  Click **Remove**, type `delete` to confirm.

### Step 3: IAM Role
1.  Go to the **IAM Console** -> **Roles**.
2.  Search for `eks-pod-identity-s3-role`.
3.  Select it and click **Delete**.

### Step 4: EKS Cluster
1.  Go back to the **EKS Clusters list**.
2.  Select your cluster and click **Delete**.
3.  *Note: Ensure you delete the Node Group first if you created one manually.*

---

## 4. Why this order matters? 🤔

*   **Permissions**: If you delete the IAM Role first, the Pod Identity Agent might go into a "Degraded" state because the linked role no longer exists.
*   **Costs**: If you delete the cluster but forget the IAM Role, you are leaving security "holes" and clutter in your AWS account.
*   **Stuck Resources**: Deleting a namespace that has active associations can sometimes lead to the namespace hanging in a `Terminating` state.

---

### End of Course! 🏆
You have successfully learned how to **Setup**, **Validate**, and **Cleanup** EKS Pod Identity!
