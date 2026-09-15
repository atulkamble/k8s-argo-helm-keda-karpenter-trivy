# Karpenter: add worker nodes

Run commands from the repository root. This lesson requires an existing EKS
cluster with Karpenter v1 installed and its AWS/IAM setup completed.
These manifests do not install the controller or create IAM roles.

1. Edit `ec2nodeclass.yaml`: replace the node role and cluster-name placeholders.
2. Ensure the intended subnets and security groups have matching discovery tags.
3. Apply the node configuration, then create capacity demand:

```bash
kubectl apply -f karpenter/ec2nodeclass.yaml
kubectl apply -f karpenter/nodepool.yaml
kubectl wait --for=condition=Ready ec2nodeclass/demo --timeout=120s
kubectl wait --for=condition=Ready nodepool/demo --timeout=120s
kubectl apply -f karpenter/capacity-demo.yaml
kubectl scale deployment capacity-demo -n default --replicas=20
kubectl get pods -n default -l app=capacity-demo -w
```

In another terminal:

```bash
kubectl get nodes -w
```

`EC2NodeClass` describes AWS settings. `NodePool` describes allowed nodes.
The demo Deployment starts at zero replicas. Scaling to 20 requests 20 CPUs
on the `demo` pool, creating demand without interfering with the webapp's KEDA
or Argo CD settings. AWS quota and available capacity still affect provisioning.
EC2 nodes incur charges; the 32-CPU pool limit is not a billing cap.

## Cleanup

```bash
kubectl delete -f karpenter/capacity-demo.yaml
kubectl delete -f karpenter/nodepool.yaml
kubectl get nodeclaims
```

Wait for the demo NodeClaims and EC2 instances to be removed, then:

```bash
kubectl delete -f karpenter/ec2nodeclass.yaml
```

Setup reference: [Karpenter getting started](https://karpenter.sh/docs/getting-started/).
