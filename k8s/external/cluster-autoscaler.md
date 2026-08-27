## Cluster Autoscaler Setup on Amazon EKS
- Cluster Autoscaler automatically adds or removes worker nodes in an EKS cluster based on workload, ensuring high availability, efficient resource utilization, and cost optimization.

**Why OIDC + IRSA?**

- The Cluster Autoscaler runs as a Pod inside Kubernetes. Pods don't have AWS credentials by default. **OIDC + IRS**A lets the Pod securely assume an IAM Role to interact with AWS Auto Scaling Groups.

### How to Install Cluster Autoscaler on EKS Using IRSA
**The breakdown is:**
1. Check/Create OIDC Provider
2. Create Custom IAM Policy for Cluster Autoscaler
3. Create IAM Role
4. Attach IAM Policy
5. Edit Trust Policy
6. Copy Role ARN
7. Download and Edit official manifest
8. Deploy Cluster Autoscaler
9. Verify

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

### Step 2: Create Custom IAM Policy

- **Go to:** IAM → Policies → Create policy → **Select:** JSON

**Paste:**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Action": [
        "autoscaling:SetDesiredCapacity",
        "autoscaling:TerminateInstanceInAutoScalingGroup"
      ],
      "Effect": "Allow",
      "Resource": "*"
    },
    {
      "Action": [
        "autoscaling:DescribeAutoScalingGroups",
        "autoscaling:DescribeAutoScalingInstances",
        "autoscaling:DescribeLaunchConfigurations",
        "autoscaling:DescribeScalingActivities",
        "autoscaling:DescribeTags",
        "ec2:DescribeLaunchTemplateVersions",
        "ec2:DescribeInstanceTypes",
        "ec2:GetInstanceTypesFromInstanceRequirements",
        "eks:DescribeNodegroup"
      ],
      "Effect": "Allow",
      "Resource": "*"
    }
  ]
}
```

- **Click:** Next

- **Policy name:** rr-app-cluster-autoscaler-policy

- **Click:** Create policy

- **Note:** AWS does not provide an AWS-managed IAM policy specifically for Cluster Autoscaler.
- **Note:** For the Amazon EBS CSI Driver, AWS provides managed policies.

### Step 3: Create IAM Role

- **Go to:** IAM → Roles → Create role
- **Choose:** Web identity
- **Identity Provider** → Select your EKS OIDC Provider
- **Audience:** Select: sts.amazonaws.com
- **Click:** Next

### Step 4: Attach Policy

- **Select:** rr-app-cluster-autoscaler-policy

- **Click:** Next

- **Role Name:** rr-app-cluster-autoscaler then **Create Role**.

### Step 5: Edit Trust Policy  *(Who is allowed to assume (use) this role?)*

- **Open:** IAM → Roles → rr-app-cluster-autoscaler → Trust Relationships → Edit trust policy

- Find: "Condition": {} or existing conditions.

*- *Change it so only:** "system:serviceaccount:kube-system:cluster-autoscaler" can use the role."

**Example:**

```json
"Condition": {
  "StringEquals": {
    "oidc.eks.ap-south-1.amazonaws.com/id/ABCD123456789:sub": "system:serviceaccount:kube-system:cluster-autoscaler",
    "oidc.eks.ap-south-1.amazonaws.com/id/ABCD123456789:aud": "sts.amazonaws.com"
  }
}
```
Save.

### Step 6: Copy Role ARN

- **Copy it:** Update ARN on ServiceAccount

---

### step 7: Download & Edit official manifest with:
- **Annotate ServiceAccount with IAM Role ARN (IRSA)**

- **Download & Open**
```bash
mkdir -p platform/cluster-autoscaler

curl -o platform/cluster-autoscaler/cluster-autoscaler.yaml \
https://raw.githubusercontent.com/kubernetes/autoscaler/master/cluster-autoscaler/cloudprovider/aws/examples/cluster-autoscaler-autodiscover.yaml
```
**Edit**

```
nano platform/cluster-autoscalercluster-autoscaler.yaml
```
1. **Add Cluster name:**
   ```bash
       --node-group-auto-discovery=asg:tag=k8s.io/cluster-autoscaler/enabled,k8s.io/cluster-autoscaler/<YOUR CLUSTER NAME>
   ``
3. **Add IRSA annotation:**
```bash
       annotations:
         eks.amazonaws.com/role-arn: <ARN>
```

**YAML**
```yaml
---
apiVersion: v1
kind: ServiceAccount
metadata:
  labels:
    k8s-addon: cluster-autoscaler.addons.k8s.io
    k8s-app: cluster-autoscaler
  name: cluster-autoscaler
  namespace: kube-system
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/rr-app-cluster-autoscaler
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cluster-autoscaler
  labels:
    k8s-addon: cluster-autoscaler.addons.k8s.io
    k8s-app: cluster-autoscaler
rules:
  - apiGroups: [""]
    resources: ["events", "endpoints"]
    verbs: ["create", "patch"]
  - apiGroups: [""]
    resources: ["pods/eviction"]
    verbs: ["create"]
  - apiGroups: [""]
    resources: ["pods/status"]
    verbs: ["update"]
  - apiGroups: [""]
    resources: ["endpoints"]
    resourceNames: ["cluster-autoscaler"]
    verbs: ["get", "update"]
  - apiGroups: [""]
    resources: ["nodes"]
    verbs: ["watch", "list", "get", "update"]
  - apiGroups: [""]
    resources:
      - "namespaces"
      - "pods"
      - "services"
      - "replicationcontrollers"
      - "persistentvolumeclaims"
      - "persistentvolumes"
    verbs: ["watch", "list", "get"]
  - apiGroups: ["extensions"]
    resources: ["replicasets", "daemonsets"]
    verbs: ["watch", "list", "get"]
  - apiGroups: ["policy"]
    resources: ["poddisruptionbudgets"]
    verbs: ["watch", "list"]
  - apiGroups: ["apps"]
    resources: ["statefulsets", "replicasets", "daemonsets"]
    verbs: ["watch", "list", "get"]
  - apiGroups: ["resource.k8s.io"]
    resources: ["deviceclasses", "resourceslices", "resourceclaims"]
    verbs: ["watch", "list", "get"]
  - apiGroups: ["storage.k8s.io"]
    resources: ["storageclasses", "csinodes", "csidrivers", "csistoragecapacities", "volumeattachments"]
    verbs: ["watch", "list", "get"]
  - apiGroups: ["batch", "extensions"]
    resources: ["jobs"]
    verbs: ["get", "list", "watch", "patch"]
  - apiGroups: ["coordination.k8s.io"]
    resources: ["leases"]
    verbs: ["create"]
  - apiGroups: ["coordination.k8s.io"]
    resourceNames: ["cluster-autoscaler"]
    resources: ["leases"]
    verbs: ["get", "update"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: cluster-autoscaler
  namespace: kube-system
  labels:
    k8s-addon: cluster-autoscaler.addons.k8s.io
    k8s-app: cluster-autoscaler
rules:
  - apiGroups: [""]
    resources: ["configmaps"]
    verbs: ["create", "list", "watch"]
  - apiGroups: [""]
    resources: ["configmaps"]
    resourceNames: ["cluster-autoscaler-status", "cluster-autoscaler-priority-expander"]
    verbs: ["delete", "get", "update", "watch"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: cluster-autoscaler
  labels:
    k8s-addon: cluster-autoscaler.addons.k8s.io
    k8s-app: cluster-autoscaler
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-autoscaler
subjects:
  - kind: ServiceAccount
    name: cluster-autoscaler
    namespace: kube-system

---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: cluster-autoscaler
  namespace: kube-system
  labels:
    k8s-addon: cluster-autoscaler.addons.k8s.io
    k8s-app: cluster-autoscaler
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: cluster-autoscaler
subjects:
  - kind: ServiceAccount
    name: cluster-autoscaler
    namespace: kube-system

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cluster-autoscaler
  namespace: kube-system
  labels:
    app: cluster-autoscaler
spec:
  replicas: 1
  selector:
    matchLabels:
      app: cluster-autoscaler
  template:
    metadata:
      labels:
        app: cluster-autoscaler
      annotations:
        prometheus.io/scrape: 'true'
        prometheus.io/port: '8085'
    spec:
      priorityClassName: system-cluster-critical
      securityContext:
        runAsNonRoot: true
        runAsUser: 65534
        fsGroup: 65534
        seccompProfile:
          type: RuntimeDefault
      serviceAccountName: cluster-autoscaler
      containers:
        - image: registry.k8s.io/autoscaling/cluster-autoscaler:v1.32.1
          name: cluster-autoscaler
          resources:
            limits:
              cpu: 100m
              memory: 600Mi
            requests:
              cpu: 100m
              memory: 600Mi
          command:
            - ./cluster-autoscaler
            - --v=4
            - --stderrthreshold=info
            - --cloud-provider=aws
            - --skip-nodes-with-local-storage=false
            - --expander=least-waste
            - --node-group-auto-discovery=asg:tag=k8s.io/cluster-autoscaler/enabled,k8s.io/cluster-autoscaler/<CLUSTER NAME>
          volumeMounts:
            - name: ssl-certs
              mountPath: /etc/ssl/certs/ca-certificates.crt # /etc/ssl/certs/ca-bundle.crt for Amazon Linux Worker Nodes
              readOnly: true
          imagePullPolicy: "Always"
          securityContext:
            allowPrivilegeEscalation: false
            capabilities:
              drop:
                - ALL
            readOnlyRootFilesystem: true
      volumes:
        - name: ssl-certs
          hostPath:
            path: "/etc/ssl/certs/ca-bundle.crt"
```
### Step 8: Deploy Cluster Autoscaler

**Apply:**
```bash
kubectl apply -f platform/cluster-autoscaler/cluster-autoscaler.yaml
```

### step 8: Verify:

```bash
kubectl get pods -n kube-system | grep cluster-autoscaler

kubectl logs -n kube-system deployment/cluster-autoscaler
```
---

## *Before applying, make sure:*

1. **OIDC provider is associated with your EKS cluster.**
2. **IRSA IAM role exists.**
   
3. **Your node group's ASG has the required Cluster Autoscaler tags.**
```sql
  k8s.io/cluster-autoscaler/enabled = true
  k8s.io/cluster-autoscaler/rr-cluster = owned
  ```
4. **The autoscaler version matches your Kubernetes/EKS version.**
