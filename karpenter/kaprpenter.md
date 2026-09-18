# Karpenter Practice

## 1. What is Karpenter?

Karpenter is a Kubernetes **node lifecycle and autoscaling** solution. It watches for unschedulable Pods and can provision suitable EC2 capacity for an EKS cluster.

```text
HPA / KEDA → Scale Pods
Karpenter  → Provision / remove Nodes
```

Basic flow:

```text
Application
     ↓
More Pods
     ↓
Existing Nodes don't have enough capacity
     ↓
Pods become Pending
     ↓
Karpenter detects Pending Pods
     ↓
Selects suitable EC2 capacity
     ↓
Creates NodeClaim
     ↓
EC2 Instance launches
     ↓
Node joins EKS
     ↓
Pods become Running
```

[Karpenter Documentation](https://karpenter.sh/docs/?utm_source=chatgpt.com)

---

# 2. Points to Remember

```text
1. Karpenter works primarily at the Node level.

2. HPA:
   Scales Pods based on metrics.

3. KEDA:
   Scales workloads based on events/metrics.

4. Karpenter:
   Provisions and manages Nodes.

5. Karpenter watches unschedulable Pods.

6. Pod resource requests are important:
   CPU
   Memory

7. Main Karpenter resources:

   NodePool
       ↓
   EC2NodeClass
       ↓
   NodeClaim
       ↓
   EC2 Instance
       ↓
   Kubernetes Node

8. NodePool:
   Defines what types of nodes Karpenter is allowed
   to provision.

9. EC2NodeClass:
   Defines AWS-specific configuration such as:
   - AMI
   - IAM role
   - Subnets
   - Security Groups

10. NodeClaim:
    Represents an individual capacity request/node
    created by Karpenter.

11. Karpenter supports:
    - On-Demand
    - Spot

12. Karpenter can choose suitable EC2 instance types.

13. Karpenter consolidation can remove unnecessary
    or underutilized capacity.

14. Karpenter Controller should run on existing
    stable capacity.

15. Installing Helm alone does not automatically
    create all required AWS permissions.
```

---

# 3. Why CloudFormation?

**CloudFormation is not Karpenter.**

Karpenter has two major sides:

```text
             Karpenter Setup

        AWS Side          Kubernetes Side
           |                    |
           v                    v
    IAM Permissions          Helm
    Node IAM Role              |
    Controller Role            v
    AWS Resources        Karpenter Controller
           |                    |
           +---------+----------+
                     |
                     v
                 Karpenter
```

CloudFormation is simply a convenient way to create the **AWS-side prerequisites**.

For example:

```text
CloudFormation
      ↓
Create IAM Roles
      ↓
Create IAM Policies
      ↓
Configure permissions
      ↓
Allow Karpenter to use AWS APIs
```

Karpenter needs AWS permissions because it may need to:

```text
Discover EC2 instance types
Discover subnets
Discover security groups
Launch EC2 instances
Pass IAM roles to EC2
Manage instance capacity
Terminate unnecessary instances
```

[Karpenter CloudFormation Reference](https://karpenter.sh/docs/reference/cloudformation/?utm_source=chatgpt.com)

### Is CloudFormation mandatory?

**No.**

You have two approaches:

```text
OPTION 1 – CloudFormation

CloudFormation
     ↓
AWS prerequisites
     ↓
Helm
     ↓
Karpenter


OPTION 2 – Manual

Create IAM roles
Create IAM policies
Configure Pod Identity / IRSA
Configure node permissions
Configure discovery
     ↓
Helm
     ↓
Karpenter
```

For a **basic classroom lab**, CloudFormation is easier because AWS/IAM setup is otherwise the longest part.

---

# 4. Prerequisites

Your EKS cluster already exists.

Check required tools:

```bash
aws --version
kubectl version --client
helm version
```

Verify AWS:

```bash
aws sts get-caller-identity
```

Verify cluster:

```bash
kubectl get nodes
```

```bash
kubectl get pods -A
```

---

# 5. Set Environment Variables

Change according to your environment:

```bash
export CLUSTER_NAME="mycluster"
export AWS_DEFAULT_REGION="us-east-1"
export KARPENTER_NAMESPACE="kube-system"

export AWS_ACCOUNT_ID="$(aws sts get-caller-identity \
  --query Account \
  --output text)"
```

Set the Karpenter version you have chosen/tested:

```bash
export KARPENTER_VERSION="<KARPENTER_VERSION>"
```

Check:

```bash
echo $CLUSTER_NAME
echo $AWS_DEFAULT_REGION
echo $AWS_ACCOUNT_ID
echo $KARPENTER_VERSION
```

---

# 6. AWS Prerequisites Using CloudFormation

Download the bootstrap template corresponding to your selected Karpenter version.

Follow the current official template instructions here:

[Karpenter Getting Started](https://karpenter.sh/docs/getting-started/?utm_source=chatgpt.com)

Typical pattern:

```bash
curl -fsSL \
https://raw.githubusercontent.com/aws/karpenter-provider-aws/v${KARPENTER_VERSION}/website/content/en/preview/getting-started/getting-started-with-karpenter/cloudformation.yaml \
-o cloudformation.yaml
```

Check:

```bash
ls -lh cloudformation.yaml
```

Deploy:

```bash
aws cloudformation deploy \
  --stack-name "Karpenter-${CLUSTER_NAME}" \
  --template-file cloudformation.yaml \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameter-overrides \
  "ClusterName=${CLUSTER_NAME}"
```

Check:

```bash
aws cloudformation describe-stacks \
  --stack-name "Karpenter-${CLUSTER_NAME}" \
  --query "Stacks[0].StackStatus"
```

Expected:

```text
CREATE_COMPLETE
```

### What did we accomplish?

```text
CloudFormation
      ↓
AWS-side prerequisites
      ↓
IAM roles / policies / permissions
      ↓
Karpenter can interact with AWS
```

---

# 7. Configure Subnet Discovery

Karpenter needs to know which subnets it may use.

Get EKS subnets:

```bash
aws eks describe-cluster \
  --name $CLUSTER_NAME \
  --query "cluster.resourcesVpcConfig.subnetIds" \
  --output text
```

Store:

```bash
SUBNET_IDS=$(aws eks describe-cluster \
  --name $CLUSTER_NAME \
  --query "cluster.resourcesVpcConfig.subnetIds" \
  --output text)
```

Tag them:

```bash
for subnet in $SUBNET_IDS; do
  aws ec2 create-tags \
    --resources $subnet \
    --tags Key=karpenter.sh/discovery,Value=$CLUSTER_NAME
done
```

Verify:

```bash
aws ec2 describe-subnets \
  --filters \
  "Name=tag:karpenter.sh/discovery,Values=$CLUSTER_NAME"
```

---

# 8. Configure Security Group Discovery

Get cluster security group:

```bash
CLUSTER_SG=$(aws eks describe-cluster \
  --name $CLUSTER_NAME \
  --query "cluster.resourcesVpcConfig.clusterSecurityGroupId" \
  --output text)
```

Check:

```bash
echo $CLUSTER_SG
```

Tag:

```bash
aws ec2 create-tags \
  --resources $CLUSTER_SG \
  --tags Key=karpenter.sh/discovery,Value=$CLUSTER_NAME
```

---

# 9. Install Karpenter Using Helm

CloudFormation prepared the AWS side.

Now Helm installs the **Karpenter Controller inside EKS**.

```text
CloudFormation
     ↓
AWS Prerequisites

       +

Helm
     ↓
Karpenter Controller
```

Install:

```bash
helm registry logout public.ecr.aws
```

```bash
helm upgrade --install karpenter \
  oci://public.ecr.aws/karpenter/karpenter \
  --version "${KARPENTER_VERSION}" \
  --namespace "${KARPENTER_NAMESPACE}" \
  --create-namespace \
  --set "settings.clusterName=${CLUSTER_NAME}" \
  --wait
```

The exact controller identity configuration—Pod Identity or IRSA—must also match the AWS prerequisites you created.

---

# 10. Verify Installation

Check Helm:

```bash
helm list -n kube-system
```

Check Karpenter:

```bash
kubectl get pods -n kube-system | grep karpenter
```

Check deployment:

```bash
kubectl get deployment karpenter -n kube-system
```

Check CRDs:

```bash
kubectl get crd | grep karpenter
```

You should see resources for:

```text
NodePool
NodeClaim
EC2NodeClass
```

Logs:

```bash
kubectl logs \
  -n kube-system \
  -l app.kubernetes.io/name=karpenter \
  -c controller \
  -f
```

---

# 11. Create EC2NodeClass + NodePool

Create:

```bash
nano karpenter.yaml
```

Add:

```yaml
apiVersion: karpenter.k8s.aws/v1
kind: EC2NodeClass
metadata:
  name: default

spec:

  role: "KarpenterNodeRole-mycluster"

  amiSelectorTerms:
    - alias: al2023@latest

  subnetSelectorTerms:
    - tags:
        karpenter.sh/discovery: mycluster

  securityGroupSelectorTerms:
    - tags:
        karpenter.sh/discovery: mycluster

---
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: default

spec:

  template:
    spec:

      nodeClassRef:
        group: karpenter.k8s.aws
        kind: EC2NodeClass
        name: default

      requirements:

        - key: kubernetes.io/arch
          operator: In
          values:
            - amd64

        - key: kubernetes.io/os
          operator: In
          values:
            - linux

        - key: karpenter.sh/capacity-type
          operator: In
          values:
            - on-demand

        - key: karpenter.k8s.aws/instance-category
          operator: In
          values:
            - c
            - m
            - r

        - key: karpenter.k8s.aws/instance-generation
          operator: Gt
          values:
            - "2"

  limits:
    cpu: 100

  disruption:
    consolidationPolicy: WhenEmptyOrUnderutilized
    consolidateAfter: 1m
```

Replace:

```text
mycluster
```

with your actual cluster name.

For a repeatable production-style setup, prefer a tested/pinned AMI alias rather than automatically following `@latest`.

---

# 12. Apply Karpenter Configuration

```bash
kubectl apply -f karpenter.yaml
```

Check:

```bash
kubectl get nodepools
```

```bash
kubectl get ec2nodeclasses
```

Detailed:

```bash
kubectl describe nodepool default
```

```bash
kubectl describe ec2nodeclass default
```

---

# 13. Create Test Workload

Create:

```bash
nano inflate.yaml
```

Add:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: inflate

spec:
  replicas: 0

  selector:
    matchLabels:
      app: inflate

  template:
    metadata:
      labels:
        app: inflate

    spec:
      containers:

        - name: inflate
          image: public.ecr.aws/eks-distro/kubernetes/pause:3.10

          resources:
            requests:
              cpu: "1"
              memory: "1Gi"
```

Apply:

```bash
kubectl apply -f inflate.yaml
```

---

# 14. Check Environment Before Scaling

```bash
kubectl get nodes
```

```bash
kubectl get pods
```

```bash
kubectl get nodeclaims
```

Example:

```text
Existing EKS Nodes   → 2
Karpenter Nodes      → 0
NodeClaims           → 0
```

---

# 15. Generate Load

Scale:

```bash
kubectl scale deployment inflate --replicas=10
```

Watch:

```bash
kubectl get pods -w
```

Check only Pending Pods:

```bash
kubectl get pods \
  --field-selector=status.phase=Pending
```

Concept:

```text
10 Pods
   ↓
Existing Nodes
   ↓
Insufficient CPU / RAM
   ↓
Some Pods Pending
```

---

# 16. Watch Karpenter

### Terminal 1 – Pods

```bash
kubectl get pods -w
```

### Terminal 2 – Nodes

```bash
kubectl get nodes -w
```

### Terminal 3 – NodeClaims

```bash
kubectl get nodeclaims -w
```

### Terminal 4 – Karpenter Logs

```bash
kubectl logs \
  -n kube-system \
  -l app.kubernetes.io/name=karpenter \
  -c controller \
  -f
```

Expected:

```text
Pending Pods
     ↓
Karpenter
     ↓
Evaluate Pod requirements
     ↓
NodePool
     ↓
EC2NodeClass
     ↓
Create NodeClaim
     ↓
Launch EC2
     ↓
Register Node
     ↓
Schedule Pods
```

---

# 17. Verify Autoscaling

```bash
kubectl get nodes -o wide
```

```bash
kubectl get nodeclaims
```

```bash
kubectl get pods -o wide
```

Describe NodeClaim:

```bash
kubectl describe nodeclaim
```

The important relationship is:

```text
NodePool
    |
    v
EC2NodeClass
    |
    v
NodeClaim
    |
    v
EC2 Instance
    |
    v
Kubernetes Node
    |
    v
Pods
```

---

# 18. Test Scale Down

Scale workload back to zero:

```bash
kubectl scale deployment inflate --replicas=0
```

Watch:

```bash
kubectl get pods
```

```bash
kubectl get nodes -w
```

```bash
kubectl get nodeclaims -w
```

With consolidation configured, Karpenter can remove capacity that is no longer required.

---

# 19. Cleanup Test Workload

```bash
kubectl delete deployment inflate
```

Delete the practice NodePool/NodeClass if required:

```bash
kubectl delete -f karpenter.yaml
```

Do not delete the CloudFormation stack unless you want to remove the AWS-side Karpenter prerequisites too.

---

# 20. Important Commands

```bash
# Karpenter Controller
kubectl get pods -n kube-system | grep karpenter

# NodePools
kubectl get nodepools

# EC2NodeClasses
kubectl get ec2nodeclasses

# NodeClaims
kubectl get nodeclaims

# Nodes
kubectl get nodes -o wide

# Pods
kubectl get pods -o wide

# Pending Pods
kubectl get pods \
  --field-selector=status.phase=Pending

# NodePool
kubectl describe nodepool default

# EC2NodeClass
kubectl describe ec2nodeclass default

# NodeClaims
kubectl describe nodeclaim

# Karpenter Logs
kubectl logs \
  -n kube-system \
  -l app.kubernetes.io/name=karpenter \
  -c controller \
  -f
```

# Final Practice Flow

```text
Existing EKS
     ↓
Verify Cluster
     ↓
Set Variables
     ↓
Configure AWS Prerequisites
     ↓
CloudFormation
(IAM roles/policies etc.)
     ↓
Configure Discovery
(Subnets + Security Groups)
     ↓
Configure Controller AWS Identity
     ↓
Install Karpenter using Helm
     ↓
Verify Controller
     ↓
Create EC2NodeClass
     ↓
Create NodePool
     ↓
Deploy Test Workload
     ↓
Scale to 10 Pods
     ↓
Pods become Pending
     ↓
Karpenter detects demand
     ↓
NodeClaim created
     ↓
EC2 Instance launched
     ↓
Node joins EKS
     ↓
Pods become Running
     ↓
Scale Pods to 0
     ↓
Karpenter Consolidation
     ↓
Unnecessary Node removed
```

### One-line concept to remember

```text
HPA/KEDA → "How many Pods do I need?"

Karpenter → "What Nodes do I need to run those Pods?"

CloudFormation → "Give Karpenter the AWS-side resources/permissions it needs."

Helm → "Install the Karpenter controller inside Kubernetes."
```

For the lab, **CloudFormation is a convenience, not a Karpenter requirement**—the same AWS prerequisites can be created manually. The important learning sequence is **AWS permissions → Helm controller → EC2NodeClass → NodePool → Pending Pods → NodeClaim → EC2 node → consolidation**.
