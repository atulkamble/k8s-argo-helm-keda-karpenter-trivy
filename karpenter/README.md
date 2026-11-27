# Karpenter on EKS

Karpenter watches for unschedulable Pods and creates NodeClaims for eligible EC2 capacity. The NodePool defines scheduling constraints and limits. The EC2NodeClass selects the AMI, node IAM role, subnets and security groups.

Run from the repository root against an EKS practice cluster with the Karpenter v1 controller and AWS provider installed. The controller needs existing compute to run on. The node role must be authorized to join EKS. The selected subnets and security groups must have the discovery tag shown in the manifest.

## Create a lab NodePool

Replace both placeholders in `karpenter/ec2nodeclass.yaml` with the existing IAM node role **name** and the cluster's discovery tag value. The manifest uses the latest EKS-optimized AL2023 AMI alias for the demo.

```bash
kubectl config current-context
kubectl api-resources --api-group=karpenter.sh
kubectl api-resources --api-group=karpenter.k8s.aws
kubectl apply -f karpenter/ec2nodeclass.yaml
kubectl wait --for=condition=Ready ec2nodeclass/classroom-demo --timeout=3m
kubectl apply -f karpenter/nodepool.yaml
kubectl wait --for=condition=Ready nodepool/classroom-demo --timeout=3m
```

## Trigger provisioning

```bash
kubectl create namespace webapp-lab --dry-run=client -o yaml | kubectl apply -f -
kubectl apply -f karpenter/capacity-demo.yaml
kubectl scale deployment capacity-demo -n webapp-lab --replicas=2
kubectl get pods -n webapp-lab -l app=capacity-demo -o wide -w
```

In another terminal, observe provisioning and check that the workload is ready.

```bash
kubectl get nodeclaims,nodes -l karpenter.sh/nodepool=classroom-demo
kubectl rollout status deployment/capacity-demo -n webapp-lab --timeout=10m
```

The workload selects the lab NodePool and tolerates its dedicated taint. Because a fresh pool has no nodes, even two Pods create demand. Look for Pending Pods and a NodeClaim, then Ready nodes and Running Pods. If Pods stay Pending, inspect their scheduling events and the NodeClaim conditions. AWS quota, EC2 capacity and node registration affect the result.

## Cleanup

```bash
kubectl delete -f karpenter/capacity-demo.yaml --ignore-not-found
kubectl delete -f karpenter/nodepool.yaml --ignore-not-found --timeout=10m
kubectl get nodeclaims,nodes -l karpenter.sh/nodepool=classroom-demo
```

Wait until the listing is empty. If any lab resources remain, rerun the listing before deleting the class.

```bash
kubectl delete -f karpenter/ec2nodeclass.yaml --ignore-not-found --timeout=5m
kubectl get nodeclaims,nodes -l karpenter.sh/nodepool=classroom-demo
```

Confirm the lab instances have terminated in EC2. The NodePool CPU limit constrains provisioned capacity. EC2 charges depend on the instances launched and their runtime.

[Karpenter setup](https://karpenter.sh/docs/getting-started/) · [NodePools](https://karpenter.sh/docs/concepts/nodepools/)
