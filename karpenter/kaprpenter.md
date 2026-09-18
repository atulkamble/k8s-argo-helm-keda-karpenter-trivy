Since your **EKS cluster already exists**, skip EKS creation. This is a focused **Karpenter basic practice**. Current Karpenter uses `NodePool`, `EC2NodeClass`, and `NodeClaim`; it watches unschedulable Pods and provisions suitable EC2 capacity. ([Karpenter][1])

## Karpenter – Basic Practice on Existing EKS

### Points to Remember

```text
1. Karpenter is used for Kubernetes NODE autoscaling.

2. HPA/KEDA → scale Pods.
   Karpenter → provisions/removes Nodes.

3. Karpenter watches for Pods that Kubernetes cannot schedule.

4. When Pods become Pending because of insufficient resources,
   Karpenter evaluates their requirements.

5. Karpenter can automatically select suitable EC2 instance types.

6. Important Karpenter resources:

   NodePool
      |
      └── EC2NodeClass
              |
              └── NodeClaim
                      |
                      └── EC2 Instance / Kubernetes Node

7. NodePool:
   Defines what type of capacity Karpenter is allowed to provision.

8. EC2NodeClass:
   Defines AWS-specific configuration:
   - AMI
   - IAM role
   - Subnets
   - Security Groups

9. NodeClaim:
   Represents an individual capacity request/node created by Karpenter.

10. Karpenter controller itself must run on existing stable capacity.
    Do not depend on Karpenter to create the node required to run
    the Karpenter controller.

11. CPU/Memory requests are important.
    Karpenter uses Pod scheduling requirements when selecting capacity.

12. Karpenter can use:
    - On-Demand
    - Spot

13. Node consolidation can remove unnecessary/underutilized nodes.

14. At least one NodePool is required for Karpenter to provision nodes.
```

The current Karpenter documentation confirms that a NodePool defines provisioning constraints and each AWS NodePool references an `EC2NodeClass`. ([Karpenter][2])

## Practice Flow

```text
Existing EKS Cluster
        |
        v
Install Karpenter
        |
        v
Create EC2NodeClass
        |
        v
Create NodePool
        |
        v
Deploy Test Application
        |
        v
Increase Replicas
        |
        v
Existing Nodes become full
        |
        v
Pods become Pending
        |
        v
Karpenter detects Pending Pods
        |
        v
Creates NodeClaim
        |
        v
Launches EC2 Instance
        |
        v
Node joins EKS
        |
        v
Pending Pods become Running
        |
        v
Scale workload down
        |
        v
Karpenter can consolidate capacity
```

## Step 1 — Verify Existing EKS

```bash
kubectl get nodes

kubectl get pods -A

aws sts get-caller-identity
```

Get cluster information:

```bash
aws eks describe-cluster \
  --name mycluster \
  --region us-east-1
```

Set variables:

```bash
export CLUSTER_NAME="mycluster"
export AWS_DEFAULT_REGION="us-east-1"
export AWS_ACCOUNT_ID="$(aws sts get-caller-identity --query Account --output text)"
```

Change `mycluster` and region if necessary.

## Step 2 — Karpenter AWS Prerequisites

For an **existing cluster**, Karpenter still needs AWS-side permissions/resources. The official bootstrap CloudFormation is specifically useful for setting up permissions needed when adding Karpenter to an existing cluster. ([Karpenter][3])

Use the current Karpenter version rather than hard-coding an old release:

```bash
export KARPENTER_VERSION="<CURRENT_VERSION>"
```

[Karpenter installation documentation](https://karpenter.sh/docs/getting-started/?utm_source=chatgpt.com)

Download the corresponding CloudFormation template:

```bash
curl -fsSL \
https://raw.githubusercontent.com/aws/karpenter-provider-aws/v${KARPENTER_VERSION}/website/content/en/preview/getting-started/getting-started-with-karpenter/cloudformation.yaml \
-o karpenter-cloudformation.yaml
```

Deploy:

```bash
aws cloudformation deploy \
  --stack-name "Karpenter-${CLUSTER_NAME}" \
  --template-file karpenter-cloudformation.yaml \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameter-overrides "ClusterName=${CLUSTER_NAME}"
```

This bootstrap is important because installing only the Helm chart is **not enough** for an existing EKS cluster.

## Step 3 — Tag Subnets and Security Groups

Karpenter needs to discover where it can launch EC2 instances.

Typical discovery tag:

```text
karpenter.sh/discovery = mycluster
```

Check VPC/subnets:

```bash
aws eks describe-cluster \
  --name $CLUSTER_NAME \
  --query "cluster.resourcesVpcConfig"
```

You can inspect subnet tags with:

```bash
aws ec2 describe-subnets \
  --filters "Name=tag:karpenter.sh/discovery,Values=${CLUSTER_NAME}"
```

And security groups:

```bash
aws ec2 describe-security-groups \
  --filters "Name=tag:karpenter.sh/discovery,Values=${CLUSTER_NAME}"
```

Your NodeClass can then discover these resources through the tag. ([Karpenter][4])

## Step 4 — Install Karpenter Using Helm

Karpenter is distributed through its OCI Helm chart. ([Karpenter][4])

```bash
helm registry logout public.ecr.aws
```

Install:

```bash
helm upgrade --install karpenter \
  oci://public.ecr.aws/karpenter/karpenter \
  --version "${KARPENTER_VERSION}" \
  --namespace kube-system \
  --create-namespace \
  --set "settings.clusterName=${CLUSTER_NAME}" \
  --wait
```

Verify:

```bash
kubectl get pods -n kube-system | grep karpenter
```

Check:

```bash
kubectl get deployment karpenter -n kube-system
```

Logs:

```bash
kubectl logs \
  -n kube-system \
  -l app.kubernetes.io/name=karpenter \
  -f
```

## Step 5 — Create EC2NodeClass + NodePool

Create:

```bash
nano karpenter.yaml
```

Basic example:

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

with your actual cluster name. For a repeatable lab, pinning a tested AL2023 AMI alias/version is preferable to `@latest`; the current getting-started examples use a versioned AL2023 alias. ([Karpenter][4])

Apply:

```bash
kubectl apply -f karpenter.yaml
```

Verify:

```bash
kubectl get nodepools

kubectl get ec2nodeclasses

kubectl describe nodepool default

kubectl describe ec2nodeclass default
```

## Step 6 — Create Test Workload

Create:

```bash
nano inflate.yaml
```

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

Check:

```bash
kubectl get deployment inflate
```

## Step 7 — Check Nodes Before Scaling

```bash
kubectl get nodes
```

For more information:

```bash
kubectl get nodes -o wide
```

Keep this running:

```bash
kubectl get nodes -w
```

## Step 8 — Generate Demand

Scale:

```bash
kubectl scale deployment inflate --replicas=10
```

Watch:

```bash
kubectl get pods -w
```

Check pending Pods:

```bash
kubectl get pods \
  --field-selector=status.phase=Pending
```

If existing nodes don't have sufficient capacity, some Pods should become Pending.

## Step 9 — Watch Karpenter

New terminal:

```bash
kubectl get nodeclaims -w
```

Another terminal:

```bash
kubectl get nodes -w
```

Karpenter logs:

```bash
kubectl logs \
  -n kube-system \
  -l app.kubernetes.io/name=karpenter \
  -f
```

Expected flow:

```text
10 Pods requested
      |
      v
Existing Nodes
      |
      v
Insufficient CPU/RAM
      |
      v
Pending Pods
      |
      v
Karpenter
      |
      v
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
New Kubernetes Node
      |
      v
Pods Running
```

Check:

```bash
kubectl get nodeclaims
```

```bash
kubectl get nodes
```

```bash
kubectl get pods -o wide
```

You should be able to see which Pods were placed on the Karpenter-created node.

## Step 10 — Scale Down

```bash
kubectl scale deployment inflate --replicas=0
```

Check:

```bash
kubectl get pods
```

Watch nodes:

```bash
kubectl get nodes -w
```

Because the example NodePool uses:

```yaml
disruption:
  consolidationPolicy: WhenEmptyOrUnderutilized
  consolidateAfter: 1m
```

Karpenter can consolidate unnecessary capacity after the workload disappears. ([Karpenter][4])

## Step 11 — Useful Commands

```bash
# Karpenter
kubectl get pods -n kube-system | grep karpenter

# NodePool
kubectl get nodepools

# EC2NodeClass
kubectl get ec2nodeclasses

# NodeClaims
kubectl get nodeclaims

# Nodes
kubectl get nodes

# Detailed nodes
kubectl get nodes -o wide

# Pods
kubectl get pods -o wide

# Pending pods
kubectl get pods --field-selector=status.phase=Pending

# Describe NodePool
kubectl describe nodepool default

# Describe NodeClass
kubectl describe ec2nodeclass default

# Describe NodeClaim
kubectl describe nodeclaim

# Karpenter logs
kubectl logs \
  -n kube-system \
  -l app.kubernetes.io/name=karpenter \
  -f
```

## Basic Practice Summary

```text
STEP 1  → Verify existing EKS
STEP 2  → Configure Karpenter AWS/IAM prerequisites
STEP 3  → Configure subnet/security-group discovery
STEP 4  → Install Karpenter with Helm
STEP 5  → Create EC2NodeClass
STEP 6  → Create NodePool
STEP 7  → Deploy test workload
STEP 8  → Scale workload to 10 Pods
STEP 9  → Observe Pending Pods
STEP 10 → Watch NodeClaim
STEP 11 → Watch new EC2/Kubernetes Node
STEP 12 → Verify Pods become Running
STEP 13 → Scale workload to 0
STEP 14 → Observe Karpenter consolidation
```

**Most important concept for the lab:**

```text
Pod Autoscaling                         Node Autoscaling

HPA / KEDA                             Karpenter
     |                                     |
     v                                     v
More Pods --------------------------> Pending Pods
                                          |
                                          v
                                      Karpenter
                                          |
                                          v
                                     More EC2 Nodes
```

[Karpenter official documentation](https://karpenter.sh/docs/?utm_source=chatgpt.com)

[1]: https://karpenter.sh/docs/?utm_source=chatgpt.com "Documentation | Karpenter"
[2]: https://karpenter.sh/docs/concepts/nodepools/?utm_source=chatgpt.com "NodePools | Karpenter"
[3]: https://karpenter.sh/docs/reference/cloudformation/?utm_source=chatgpt.com "CloudFormation | Karpenter"
[4]: https://karpenter.sh/v1.12/getting-started/getting-started-with-karpenter/?utm_source=chatgpt.com "Getting Started with Karpenter | Karpenter"
