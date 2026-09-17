## Karpenter — Very Basic Practice on Amazon EKS

Amazon Web Services **Karpenter** is a Kubernetes node autoscaler. It watches for **unschedulable Pods**, provisions suitable EC2 capacity, and can remove unnecessary nodes later.

![Image](https://images.openai.com/static-rsc-4/c8n_10zr8cPOZn8HATG35KzVyQi8w-Tz18Br82uXhsT2VKVH-Vtfgy1yoY66E0Y46BHm_H_5JQ4Vso7VHAleO11vqpdP7gr14KkVA5TBr9fG94UeFBu-OMJMd7mSr5EixbKQJQC23cnX4FnY0_p2GI8rCoCN2fgs-SnX1JmIenkO0GPudi3fq7Wk_DAHPx6u?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/cHmIcSQi7sDt4a71MumBW2ZbKQ8ogW3RbNIaAVwqzWIcqvBcg30A3UPpL1A7iGdUuYs99GyfkyF38XPweJx7eWJSU6jsCFb_duDVbFWJL6Jw7BK1VcEGk0vGwy2-g99exyYhKHNvQP6cR-t0tt-88zp2in1M01s--UxngsYggnEUWatUqNv7ImyJasTQ5Dft?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/cVyVSh0whPT51197arEO9q0H0vEzkhEHhO3bM9E_vwcrPbyYR_orN2id4VBHr-76b1uISFwrOqI1ZdXEOBVHJfVPvKkbyU8ED5yg5m5EVItV7STeNknBRQBJu-YGydTBDm8UfCNXgdg6JXcZhf5z6vA9gvhmIN_N7Ho3o5tnubXv-SXGfpisbfMX0eEZdRb0?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/ri4aPXG6q1ANoB0bm3x2b12ohnphWz7pMCMS6YMqKNFq7Q9oURsZVn9-EP4ZOgqBdBr7vdUO0y-UPWsrPUsCEyn1ZzSplcnpZiWPpGMeduLs3WKzEud-lzdEbOzQgs5RbSUXmVcwpI4FG3yxK3Safu02AKVq4H_XNC5_rOgfCoWOZj6NoOWMIcsVHfvJTHrl?purpose=fullsize)

### 1. Simple Architecture

```text
Developer
    |
    v
Deployment
    |
    v
Kubernetes Pods
    |
    | Pods cannot be scheduled
    v
Karpenter
    |
    v
AWS EC2
    |
    v
New Worker Node
    |
    v
Pods Scheduled
```

### 2. Prerequisites

For a basic lab:

```bash
aws --version
kubectl version --client
eksctl version
helm version
```

You need an EKS cluster and working `kubectl` access.

For example:

```bash
eksctl create cluster \
  --name mycluster \
  --region us-east-1 \
  --nodegroup-name mynodes \
  --node-type t3.medium \
  --nodes 2 \
  --managed
```

Configure `kubectl`:

```bash
aws eks update-kubeconfig \
  --name mycluster \
  --region us-east-1
```

Verify:

```bash
kubectl get nodes
kubectl get pods -A
```

### 3. Important Karpenter Objects

The two main resources to understand are:

```text
NodePool
   |
   v
EC2NodeClass
   |
   v
AWS EC2 Instance
```

**NodePool** defines Kubernetes-side requirements such as instance categories, architecture, capacity type, limits, and disruption behavior.

**EC2NodeClass** defines AWS-specific configuration such as AMI family, IAM role, subnets, and security groups.

### 4. Install Karpenter

Karpenter installation requires AWS IAM permissions and EKS integration, so don't treat it as only a Helm installation.

The official installation guide is the safest starting point because the Helm/chart version and required IAM configuration change between releases:

[Karpenter Getting Started Guide](https://karpenter.sh/docs/getting-started/getting-started-with-karpenter/?utm_source=chatgpt.com)

The installation itself uses the Karpenter OCI Helm chart in a pattern similar to:

```bash
helm upgrade --install karpenter \
  oci://public.ecr.aws/karpenter/karpenter \
  --namespace karpenter \
  --create-namespace \
  --version <KARPENTER_VERSION> \
  ...
```

After completing the IAM and Helm steps from the official guide:

```bash
kubectl get pods -n karpenter
```

Expected conceptually:

```text
NAME                         READY   STATUS
karpenter-xxxxxxxxxx-xxxxx   1/1     Running
```

### 5. Create an EC2NodeClass

A simplified example:

```yaml
apiVersion: karpenter.k8s.aws/v1
kind: EC2NodeClass
metadata:
  name: default

spec:
  amiSelectorTerms:
    - alias: al2023@latest

  role: "KarpenterNodeRole-mycluster"

  subnetSelectorTerms:
    - tags:
        karpenter.sh/discovery: mycluster

  securityGroupSelectorTerms:
    - tags:
        karpenter.sh/discovery: mycluster
```

Save:

```bash
nano ec2nodeclass.yaml
```

Apply:

```bash
kubectl apply -f ec2nodeclass.yaml
```

Check:

```bash
kubectl get ec2nodeclass
```

### 6. Create a NodePool

Create:

```bash
nano nodepool.yaml
```

Example:

```yaml
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

        - key: karpenter.sh/capacity-type
          operator: In
          values:
            - on-demand

      expireAfter: 720h

  limits:
    cpu: 20

  disruption:
    consolidationPolicy: WhenEmptyOrUnderutilized
    consolidateAfter: 1m
```

Apply:

```bash
kubectl apply -f nodepool.yaml
```

Verify:

```bash
kubectl get nodepool
kubectl describe nodepool default
```

### 7. Create Workload

Now create enough Pods to require additional compute capacity.

```bash
kubectl create deployment nginx --image=nginx
```

Scale it:

```bash
kubectl scale deployment nginx --replicas=10
```

Watch Pods:

```bash
kubectl get pods -w
```

In another terminal:

```bash
kubectl get nodes -w
```

The important behavior to observe is:

```text
10 Pods requested
      ↓
Existing nodes lack capacity
      ↓
Pods become Pending
      ↓
Karpenter detects scheduling requirements
      ↓
EC2 capacity launched
      ↓
Node joins EKS
      ↓
Pending Pods scheduled
```

### 8. Check Karpenter

Check NodePools:

```bash
kubectl get nodepool
```

Check NodeClaims:

```bash
kubectl get nodeclaims
```

Check nodes:

```bash
kubectl get nodes
```

Check which node each Pod is running on:

```bash
kubectl get pods -o wide
```

Karpenter logs are also very useful:

```bash
kubectl logs -n karpenter \
  -l app.kubernetes.io/name=karpenter \
  --tail=100
```

### 9. Test Scale Down

Now remove the workload:

```bash
kubectl scale deployment nginx --replicas=0
```

Watch:

```bash
kubectl get nodes -w
```

With consolidation configured correctly, Karpenter can identify unnecessary capacity and terminate the nodes it provisioned.

### 10. Points to Remember

```text
HPA
 ↓
Scales Pods

Karpenter
 ↓
Scales Nodes
```

So a common production flow is:

```text
Traffic increases
      ↓
HPA increases Pods
      ↓
Not enough node capacity
      ↓
Pods Pending
      ↓
Karpenter detects Pods
      ↓
EC2 capacity provisioned
      ↓
Pods scheduled
```

For a **first classroom demo**, focus on only four commands:

```bash
kubectl get pods -w

kubectl get nodes -w

kubectl get nodepool

kubectl get nodeclaims
```

The key takeaway for students is: **Karpenter doesn't primarily scale a predefined node group. It responds to Pod scheduling requirements and provisions appropriate EC2 capacity for those workloads.**
