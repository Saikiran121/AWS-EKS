# Hands-On: Installing the EKS Pod Identity Agent 🛠️

This lab guide walking you through the physical installation of the **EKS Pod Identity Agent** add-on. Without this agent, the Pod Identity feature will not function.

---

## 1. Preparation & Prerequisites 📋

Before you start, ensure you have:
1.  **An EKS Cluster**: Version 1.24 or higher is required.
2.  **AWS CLI**: Updated to the latest version (v2.x).
3.  **Permissions**: Your IAM user/role must have `eks:CreateAddon` permissions.

```bash
# Set your cluster context
export CLUSTER_NAME="my-eks-cluster"
export AWS_REGION="us-east-1"

# Verify connectivity
kubectl get nodes
```

---

## 2. Installation Methods 🚀

### Method A: AWS CLI (Fastest & Standard)

1.  **Check Available Versions**:
    ```bash
    aws eks describe-addon-versions \
      --addon-name eks-pod-identity-agent \
      --kubernetes-version 1.31 # Replace with your cluster version
    ```

2.  **Install the Add-on**:
    ```bash
    aws eks create-addon \
      --cluster-name $CLUSTER_NAME \
      --addon-name eks-pod-identity-agent
    ```

### Method B: AWS Management Console

1.  Open the **Amazon EKS Console**.
2.  Select **Clusters** and click on your cluster name.
3.  Click the **Add-ons** tab.
4.  Click **Get more add-ons**.
5.  Check **EKS Pod Identity Agent**.
6.  Click **Next**, then **Create**.

### Method C: Terraform (Production Automation)

Add this block to your EKS module to automate the installation:

```hcl
resource "aws_eks_addon" "pod_identity_agent" {
  cluster_name  = var.cluster_name
  addon_name    = "eks-pod-identity-agent"
  addon_version = "v1.3.2-eksbuild.2" # Use the latest stable version
}
```

---

## 3. Verification: Is it Healthy? ✅

Once installed, the agent runs as a **DaemonSet** (one pod per worker node).

```bash
# 1. Check add-on status via AWS
aws eks describe-addon \
  --cluster-name $CLUSTER_NAME \
  --addon-name eks-pod-identity-agent \
  --query "addon.status"

# 2. Verify Pods in Kubernetes
kubectl get pods -n kube-system -l app.kubernetes.io/name=eks-pod-identity-agent

# 3. Check for 'Running' status
kubectl get ds/eks-pod-identity-agent -n kube-system
```

---

## 4. Creating the ServiceAccount (The Identity) 🆔

A ServiceAccount is the entity that we will link to an IAM Role. Think of it as the "Username" for your pod inside the cluster.

### Option 1: Using Kubernetes YAML (Recommended)

Create a file named `my-service-account.yaml`:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: pod-identity-sa
  namespace: default
```

Apply it to your cluster:
```bash
kubectl apply -f my-service-account.yaml
```

### Option 2: Using kubectl CLI (Quick Way)

```bash
kubectl create serviceaccount pod-identity-sa --namespace default
```

### Verification
Ensure the ServiceAccount exists:
```bash
kubectl get sa pod-identity-sa -n default
```

---

## 5. The Validation Pod (AWS CLI) 🛰️

Now, let's create a Pod that uses the `amazon/aws-cli` image to verify our setup. This pod will eventually use the IAM permissions we associate with the ServiceAccount.

Create a file named `aws-cli-pod.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: aws-cli-test
  namespace: default
spec:
  # CRITICAL: Link to the ServiceAccount we created!
  serviceAccountName: pod-identity-sa
  
  containers:
    - name: aws-cli
      image: amazon/aws-cli:latest
      # Keep the pod alive for testing
      command: ["sleep", "3600"]
      resources:
        requests:
          cpu: "100m"
          memory: "128Mi"
        limits:
          cpu: "200m"
          memory: "256Mi"
```

### Deploy the Pod
```bash
kubectl apply -f aws-cli-pod.yaml
```

### Verify Environmental Injection
Once the pod is running, check if the EKS Pod Identity Agent has injected the required environment variables:
```bash
kubectl exec aws-cli-test -- env | grep AWS_CONTAINER
```
If you see `AWS_CONTAINER_CREDENTIALS_FULL_URI` and `AWS_CONTAINER_AUTHORIZATION_TOKEN`, the Agent is successfully communicating with your Pod!

---

## 6. Testing AWS Access (The "Fail-First" Step) 🚫

Before we apply the IAM Role Association, let's verify that the Pod **cannot** access S3. This confirms that Pod Identity is secure by default (Least Privilege).

### Step 1: Exec into the Pod
```bash
kubectl exec -it aws-cli-test -- sh
```

### Step 2: Try to list S3 buckets
Once inside the pod shell, run:
```bash
aws s3 ls
```

### Expected Result (ERROR)
You should see an error similar to:
`An error occurred (AccessDenied) when calling the ListBuckets operation: Access Denied`
OR
`Unable to locate credentials. You can configure credentials by running "aws configure".`

> [!NOTE]
> **Why does it fail?** Even though the Agent is injecting the environment variables, we haven't yet told AWS which IAM Role this ServiceAccount is allowed to use. AWS doesn't know who this pod is yet!

### Exit the Shell
```bash
exit
```

---

## 7. Creating the IAM Role (The AWS Identity) ☁️

For Pod Identity to work, your IAM Role must have a specific **Trust Policy** that allows the EKS service to give this role to your pods.

### The Trust Policy (`trust-policy.json`)

First, create a file named `trust-policy.json` on your local machine:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Service": "pods.eks.amazonaws.com"
            },
            "Action": [
                "sts:AssumeRole",
                "sts:TagSession"
            ]
        }
    ]
}
```

---

### Method A: Using AWS CLI (Recommended)

1.  **Create the IAM Role**:
    ```bash
    aws iam create-role \
      --role-name eks-pod-identity-s3-role \
      --assume-role-policy-document file://trust-policy.json
    ```

2.  **Attach a Policy (e.g., S3 Read Only)**:
    ```bash
    aws iam attach-role-policy \
      --role-name eks-pod-identity-s3-role \
      --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess
    ```

---

### Method B: Using AWS Management Console

1.  Navigate to the **IAM Console** -> **Roles** -> **Create role**.
2.  Select **Custom trust policy**.
3.  Paste the JSON from the **Trust Policy** section above.
4.  Click **Next**.
5.  Search for `AmazonS3ReadOnlyAccess`, check it, and click **Next**.
6.  Name the role: `eks-pod-identity-s3-role`.
7.  Click **Create role**.

---

### Verification
Ensure the role exists and has the correct trust relationship:
```bash
aws iam get-role --role-name eks-pod-identity-s3-role --query "Role.AssumeRolePolicyDocument"
```

---

## 8. The Final Glue: Pod Identity Association 🔗

This is the most important step. We now tell AWS: *"Anyone using the `pod-identity-sa` ServiceAccount in the `default` namespace is allowed to assume the `eks-pod-identity-s3-role`."*

### Why are we doing this?
Without this association, the EKS Pod Identity Agent won't know which IAM Role to give to your pod. This mapping decouples your Kubernetes code from your AWS security.

---

### Method A: Using AWS CLI (Fastest)

Run this command to create the mapping:

```bash
aws eks create-pod-identity-association \
  --cluster-name $CLUSTER_NAME \
  --namespace default \
  --service-account pod-identity-sa \
  --role-arn arn:aws:iam::<YOUR_ACCOUNT_ID>:role/eks-pod-identity-s3-role
```

---

### Method B: Using AWS Management Console

1.  Open the **Amazon EKS Console**.
2.  Select your **Cluster**.
3.  Click on the **Access** tab.
4.  Scroll down to the **Pod Identity associations** section.
5.  Click **Create**.
6.  **Namespace**: Select `default`.
7.  **Service Account**: Select `pod-identity-sa`.
8.  **IAM Role**: Select `eks-pod-identity-s3-role`.
9.  Click **Create**.

---

### Verification
Check that the association is active:
```bash
aws eks list-pod-identity-associations --cluster-name $CLUSTER_NAME
```

---

## 9. Basic Hands-On Lab: The "Victory" Test 🏆

Now that the association is created, let's go back into our pod and see the magic! 🎩✨

1.  **Exec into the test pod**:
    ```bash
    kubectl exec -it aws-cli-test -- sh
    ```

2.  **Try to list S3 buckets again**:
    ```bash
    aws s3 ls
    ```

3.  **Expected Result (SUCCESS)**:
    You should now see a list of your S3 buckets! No keys, no secrets, no annotations required.

---

## 10. Troubleshooting 🛠️

*   **Status: Degraded**: Check the node security groups. The agent needs to communicate internally on the node.
*   **Pods not starting**: Check if your nodes have enough memory/CPU. The agent is very lightweight (~50MB RAM).
*   **Version Mismatch**: Ensure your `addon-version` is compatible with your cluster's EKS version.

---

### Over to You! 🎯
Now that the agent is installed, you are ready to create **IAM Roles** and **Pod Identity Associations**!
