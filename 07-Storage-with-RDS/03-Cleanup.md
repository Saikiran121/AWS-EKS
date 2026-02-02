# Step-by-Step: RDS & EKS Resource Cleanup 🧹☁️💨

To avoid unnecessary AWS costs, it is essential to clean up all resources once you have finished testing. Follow these steps in order.

---

## 1. Kubernetes Resource Cleanup 🏗️🗑️

First, we remove all objects from the EKS cluster.

1.  **Delete All Manifests**:
    Navigate to the `07-Storage-with-RDS` directory and run:
    ```bash
    kubectl delete -f kube-manifests/
    ```
    **Effect**: This deletes the Deployment, NodePort Service, ExternalName Service, and the DB Password Secret.

2.  **Verify Removal**:
    ```bash
    kubectl get all
    kubectl get secret mysql-db-password
    ```

---

## 2. AWS Managed Service Cleanup 🗄️💨

Next, we delete the physical resources in the AWS Console.

### 2.1 Delete RDS Database
1.  Go to the **RDS Console** -> **Databases**.
2.  Select **webappdb**.
3.  Click **Actions** -> **Delete**.
4.  **Important**: Uncheck **Create final snapshot?** (unless you want to keep data, which costs money).
5.  Check **I acknowledge...** and type `delete me` in the box.
6.  Click **Delete**. 
    *   *Note: This takes about 5-10 minutes.*

### 2.2 Delete DB Subnet Group
1.  In the RDS Console, click **Subnet groups** in the left sidebar.
2.  Select `eks-rds-subnet-group`.
3.  Click **Delete**.

### 2.3 Delete DB Security Group
1.  Navigate to the **VPC Console** -> **Security Groups**.
2.  Search for `eks-rds-db-sg`.
3.  Select it and click **Actions** -> **Delete security groups**.

---

### Cleanup Complete! 🎖️🌪️
All storage and networking resources created for this section have been successfully removed. Your cloud bill is now safe!

---

## 💡 Summary of Resources Removed
*   **EKS**: 1 Deployment, 2 Services, 1 Secret.
*   **RDS**: 1 MySQL Instance, 1 Subnet Group.
*   **VPC**: 1 Security Group.
