
## EBS CSI Driver Installation and Setup on Amazon EKS

- EBS CSI Driver allows Kubernetes on EKS to create, attach, mount, resize, and manage EBS volumes automatically for Pod.

- **Why OIDC + IRSA?**

- The EBS CSI Driver runs as a Pod inside Kubernetes. Pods don't have AWS credentials by default. **OIDC + IRSA** lets the Pod securely assume an IAM Role.

### How to Install AWS EBS CSI Driver on EKS Using IRSA

**The breakdown is:**

1. Check/Create OIDC Provider
2. Create IAM Role
3. Attach AmazonEBSCSIDriverPolicy
4. Configure Trust Relationship
5. Create Role
6. Install EBS CSI Driver Add-on
7. Select IAM Role
8. Verify Installation
---

### Step 1: Check/Create OIDC Provider Exists

- **Go to:** EKS → Clusters → Overview → **Checks** the OIDC issuer URL.
- **Verify your EKS cluster has an OIDC provider associated.**

```bash
aws eks describe-cluster --name rr-app-cluster --query "cluster.identity.oidc.issuer" --output text
```
- **If you get, good.**
- **If not, create it:**

```bash
eksctl utils associate-iam-oidc-provider \
  --cluster my-cluster \
  --approve
```
- **Verify:**
- **Go to:** IAM → Identity providers
- **Note:** Without this, IRSA cannot work.

### Step 2: Create IAM Role

- **Go to:** IAM → Roles → Create role
- **Choose:** Web identity
- **Identity Provider** → Select your EKS OIDC Provider
- **Audience:** Select: sts.amazonaws.com
- **Click:** Next

### Step 3: Attach Required Policy

- **Search and attach:** AmazonEBSCSIDriverPolicy

- **Click:** Next.

### Step 4: Configure Trust Relationship

AWS needs to know which Kubernetes ServiceAccount can assume this role.

The trust policy should look similar to:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::<ACCOUNT-ID>:oidc-provider/oidc.eks.ap-south-1.amazonaws.com/id/XXXXXXXX"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "oidc.eks.ap-south-1.amazonaws.com/id/XXXXXXXX:sub": "system:serviceaccount:kube-system:ebs-csi-controller-sa"
        }
      }
    }
  ]
}

```

### Step 5: Create Role

- **Give role a name:** AmazonEKS_EBS_CSI_DriverRole

- **Create the role.**

- **Copy the role ARN**

### Step 6: Install EBS CSI Driver Add-on

- Go to: AWS Console → EKS → Your Cluster → Add-ons

- Click: Get More Add-ons


- **Select:** AWS EBS CSI Driver

**Click:** Next.

### Step 7: Select IAM Role

**You'll see:** IAM Role for Service Account

**Choose:** AmazonEKS_EBS_CSI_DriverRole

**Click:** Install.


### Step 8: Verify Installation

Check add-ons:

```bash
aws eks list-addons --cluster-name my-cluster
```

- **You should see:** aws-ebs-csi-driver

```bash
kubectl get pods -n kube-system
```

