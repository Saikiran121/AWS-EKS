# Hands-On: Installing Amazon EBS CSI Driver via Pod Identity 🛠️⛓️

This guide will walk you through the modern process of installing the **Amazon EBS CSI Driver** using **EKS Pod Identity**.

---

## 1. Prerequisites 📋

-   **EKS Cluster**: Version 1.24+
-   **EKS Pod Identity Agent**: This must be installed on your cluster first (See [02-Pod-Identity-Agent-HandsOn.md](file:///home/user/Skills/EKS/04-EKS-Pod-Identity/02-Pod-Identity-Agent-HandsOn.md)).
-   **AWS CLI**: Updated to the latest v2.x.

---

## 2. Step 1: Create the IAM Role ☁️

The CSI Driver needs permissions to call EC2 APIs to create, attach, and delete EBS volumes.

### 2.1 The Trust Policy (`trust-policy.json`)
Create a file named `ebs-csi-trust-policy.json`:

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

### Method A: Using AWS CLI (Recommended)

1.  **Create the IAM Role**:
    ```bash
    aws iam create-role \
      --role-name ebs-csi-pod-identity-role \
      --assume-role-policy-document file://ebs-csi-trust-policy.json
    ```

2.  **Attach the official AWS EBS CSI policy**:
    ```bash
    aws iam attach-role-policy \
      --role-name ebs-csi-pod-identity-role \
      --policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy
    ```

---

### Method B: Using AWS Management Console

1.  Open the **IAM Console** -> **Roles** -> **Create role**.
2.  Select **Custom trust policy** and paste the JSON from Section 2.1.
3.  Click **Next**.
4.  Search for **AmazonEBSCSIDriverPolicy**, select it, and click **Next**.
5.  Set **Role name** to `ebs-csi-pod-identity-role`.
6.  Click **Create role**.

---

## 3. Step 2: Create Pod Identity Association 🔗

We must link the IAM Role to the specific ServiceAccount used by the EBS CSI controller.
- **Namespace**: `kube-system`
- **Service Account**: `ebs-csi-controller-sa`

### Method A: Using AWS CLI

```bash
aws eks create-pod-identity-association \
  --cluster-name MyCluster \
  --namespace kube-system \
  --service-account ebs-csi-controller-sa \
  --role-arn arn:aws:iam::<YOUR_ACCOUNT_ID>:role/ebs-csi-pod-identity-role
```

---

### Method B: Using AWS Management Console

1.  Open the **Amazon EKS Console** -> Select your **Cluster**.
2.  Go to the **Access** tab.
3.  Scroll to **Pod Identity associations** and click **Create**.
4.  **Namespace**: `kube-system`.
5.  **Service Account**: `ebs-csi-controller-sa`.
6.  **IAM Role**: `ebs-csi-pod-identity-role`.
7.  Click **Create**.

---

## 4. Step 3: Install the EKS Add-on 🚀

### Method A: Using AWS CLI

```bash
aws eks create-addon \
  --cluster-name MyCluster \
  --addon-name aws-ebs-csi-driver
```

---

### Method B: Using AWS Management Console

1.  Open the **Amazon EKS Console** -> Select your **Cluster**.
2.  Go to the **Add-ons** tab.
3.  Click **Get more add-ons**.
4.  Check **Amazon EBS CSI Driver**.
5.  Click **Next**, use default settings, and click **Create**.

---

## 5. Step 4: Verification ✅

### 5.1 Check Pod Status
Wait for the controller and node pods to be `Running`:
```bash
kubectl get pods -n kube-system -l app.kubernetes.io/name=aws-ebs-csi-driver
```

### 5.2 Test Dynamic Provisioning
Create a `StorageClass` and a `PVC` to verify the driver can spin up an EBS volume.

**`test-storage.yaml`**:
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ebs-sc
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: ebs-claim
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: ebs-sc
  resources:
    requests:
      storage: 4Gi
```

**Apply and Check**:
```bash
kubectl apply -f test-storage.yaml
kubectl get pvc ebs-claim
```
> [!NOTE]
> The PVC will stay in `Pending` until you deploy a pod that uses it (because of `WaitForFirstConsumer`). This is normal!

---

## 6. Summary: Why Pod Identity? 🌟

1.  **Zero Annotations**: You don't need to manually annotate the `ebs-csi-controller-sa`.
2.  **No OIDC Management**: No need to create OIDC providers manually in IAM.
3.  **Unified Permissions**: Security teams manage the role and association in one place.
