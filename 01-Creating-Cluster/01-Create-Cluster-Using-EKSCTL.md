# Provisioning AWS EKS Cluster using eksctl

This guide walks through the step-by-step process of creating a production-ready EKS cluster using `eksctl`.

---

## Step 1: Create EKS Cluster (Control Plane Only)

First, we create the EKS Control Plane without any worker nodes. This allows us to configure networking and IAM providers before adding compute resources.

```bash
eksctl create cluster --name=eksdemo1 \
                      --region=us-east-1 \
                      --zones=us-east-1a,us-east-1b \
                      --without-nodegroup 
```

**What this does:**
- `--without-nodegroup`: Tells `eksctl` to only provision the managed Control Plane (API Server, etcd, etc.) and VPC/Subnets, but skip creating worker nodes (EC2 instances) for now.
- `--zones`: Specifies the Availability Zones for high availability.

---

## Step 2: Associate IAM OIDC Provider

To enable **IRSA (IAM Roles for Service Accounts)**, the cluster must have an OpenID Connect (OIDC) issuer URL associated with it.

```bash
eksctl utils associate-iam-oidc-provider \
    --region us-east-1 \
    --cluster eksdemo1 \
    --approve
```

**What this does:**
- Creates an IAM OIDC identity provider in AWS that trusts your cluster's OIDC issuer.
- This is the foundation for giving fine-grained AWS permissions to specific Pods (e.g., allowing a Pod to read from S3).

---

## Step 3: Create EC2 Keypair (Prerequisite)

Before creating nodes with SSH access, ensure you have an EC2 Key Pair created in the AWS Console named `kube-demo` (or update the command below with your key name).

---

## Step 4: Create Managed Node Group

Now we add the worker nodes (EC2 instances) to our cluster. We use a **Managed Node Group** so AWS handles the node lifecycle.

```bash
eksctl create nodegroup --cluster=eksdemo1 \
                       --region=us-east-1 \
                       --name=eksdemo1-ng-public1 \
                       --node-type=t3.medium \
                       --nodes=2 \
                       --nodes-min=2 \
                       --nodes-max=4 \
                       --node-volume-size=20 \
                       --ssh-access \
                       --ssh-public-key=kube-demo \
                       --managed \
                       --asg-access \
                       --external-dns-access \
                       --full-ecr-access \
                       --appmesh-access \
                       --alb-ingress-access
```

**Key Parameters Explained:**
- `--managed`: Enables AWS Managed Node Groups (automated updates and patching).
- `--ssh-access`: Enables SSH login to the worker nodes using the specified public key.
- `--asg-access`: Adds IAM permissions for the **Cluster Autoscaler** to manage the Auto Scaling Group.
- `--alb-ingress-access`: Adds IAM permissions for the **AWS Load Balancer Controller** to manage ALBs and Target Groups.
- `--external-dns-access`: Allows the cluster to update Route53 records via ExternalDNS.
- `--full-ecr-access`: Allows nodes to pull images from any Elastic Container Registry repository.

---

## Step 5: Verify Cluster and Nodes

Once the commands complete, verify the health of your cluster.

```bash
# List clusters
eksctl get cluster

# List node groups
eksctl get nodegroup --cluster=eksdemo1

# Check nodes using kubectl
kubectl get nodes -o wide
```

### Verify NodeGroup Subnets (Confirm Public Subnet)

To ensure your instances are in **Public Subnets**, you can query the subnet IDs associated with the NodeGroup and verify their properties.

```bash
# 1. Get the list of subnets used by the NodeGroup
aws eks describe-nodegroup --cluster eksdemo1 \
                           --name eksdemo1-ng-public1 \
                           --query 'nodegroup.subnets'

# 2. Check if the nodes have External IPs (Direct indicator of Public Subnet)
kubectl get nodes -o wide --label-columns=topology.kubernetes.io/zone
```

**Verification Checklist:**
1. Ensure the Node status is `Ready`.
2. **Subnet Check:** Confirm that each node has an **EXTERNAL-IP** assigned. If the External-IP is `<none>`, the nodes are likely in a private subnet or the VPC isn't configured for public IP assignment.
3. Check that the nodes are distributed across the specified Availability Zones.
4. Verify that the IAM OIDC provider is correctly associated in the AWS IAM Console.

---

## Step 6: Login to Worker Node (Optional)

If you need to debug a node or check container runtime logs from the machine directly, you can SSH into the worker nodes using the keypair specified during creation.

```bash
# 1. Set permissions for your private key (kube-demo.pem)
chmod 400 kube-demo.pem

# 2. Identify the External IP of the node
kubectl get nodes -o wide

# 3. SSH into the node
ssh -i "kube-demo.pem" ec2-user@<EXTERNAL-IP-OF-NODE>
```

**Note:** Ensure your VPC Security Group (for the nodes) allows SSH traffic (Port 22) from your IP.

---

## Step 7: Delete EKS Cluster & Cleanup

When you are done with your experiments, it is crucial to cleanup the resources to stop incurring costs.

### Deleting a Node Group (Only)

If you want to keep the Control Plane (API Server) running but remove all worker nodes (to save compute costs), use:

```bash
# Delete a specific node group
eksctl delete nodegroup --cluster=eksdemo1 \
                        --name=eksdemo1-ng-public1 \
                        --region=us-east-1
```

### Deleting the Entire Cluster

This command removes everything: Node Groups, VPC (if created by eksctl), and the Control Plane.

```bash
# Delete the entire cluster
eksctl delete cluster --name eksdemo1 --region us-east-1
```

### ⚠️ Critical Things to Know Before Deleting

1.  **Orphaned Load Balancers:** If you created Kubernetes Services of type `LoadBalancer` (which create AWS ELB/ALB/NLB), `eksctl` might not always delete them. 
    - **Best Practice:** Delete your Kubernetes deployments and services (`kubectl delete -f <manifests>`) **before** running the `eksctl delete cluster` command.
2.  **Persistent Volumes (EBS):** Any EBS volumes created dynamically via PVCs (Persistent Volume Claims) will **not** be deleted by AWS automatically. You must manually delete these volumes in the AWS EC2 Console to avoid ongoing storage charges.
3.  **CloudFormation Stacks:** `eksctl` uses CloudFormation. If the deletion fails, check the CloudFormation console to see which resource is blocking the deletion (usually a manual change you made to a security group or subnet).
4.  **Time:** Deletion typically takes 15-20 minutes as it has to gracefully terminate nodes and the control plane.
