# Amazon EKS Managed Node Group — User Guide

This guide explains how to create a managed node group in an existing Amazon EKS cluster using `eksctl`.

## 1. Prerequisites

Install and configure the following tools:

- [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)
- [kubectl](https://kubernetes.io/docs/tasks/tools/)
- [eksctl](https://eksctl.io/installation/)

You also need:

- An existing EKS cluster in `ACTIVE` status.
- An IAM node role with the required EKS worker-node permissions.
- Subnets and networking configured so worker nodes can reach the EKS API endpoint.
- AWS credentials with permission to create and manage node groups and related resources.

Check tool versions and AWS access:

```bash
aws --version
kubectl version --client
eksctl version
aws sts get-caller-identity
```

The source guide requires `eksctl` version `0.215.0` or later.

## 2. Set Your Cluster Details

Replace the example values with your own:

```bash
export CLUSTER_NAME="my-cluster"
export AWS_REGION="us-east-1"
export NODEGROUP_NAME="my-mng"
```

Confirm that the cluster exists and is active:

```bash
aws eks describe-cluster \
  --name "$CLUSTER_NAME" \
  --region "$AWS_REGION" \
  --query 'cluster.status' \
  --output text
```

Expected output:

```text
ACTIVE
```

Configure `kubectl` to connect to the cluster:

```bash
aws eks update-kubeconfig \
  --name "$CLUSTER_NAME" \
  --region "$AWS_REGION"
```

## 3. Create the Managed Node Group

Create a basic managed node group with three `m5.large` instances:

```bash
eksctl create nodegroup \
  --cluster "$CLUSTER_NAME" \
  --region "$AWS_REGION" \
  --name "$NODEGROUP_NAME" \
  --node-type m5.large \
  --nodes 3 \
  --nodes-min 2 \
  --nodes-max 4
```

This example maintains three nodes initially, with a configured minimum of two and maximum of four. Choose an instance type and scaling limits suitable for your workload and budget.

If you need a custom AMI or additional instance configuration, use an appropriately configured EC2 launch template. Decide whether to use a launch template when creating the node group; do not switch an existing node group from an automatically generated template to a custom one later.

## 4. Verify the Nodes

Wait for the node group to finish creating, then check its status:

```bash
eksctl get nodegroup \
  --cluster "$CLUSTER_NAME" \
  --region "$AWS_REGION"
```

Check Kubernetes nodes:

```bash
kubectl get nodes -o wide
```

Wait for nodes to become ready:

```bash
kubectl get nodes --watch
```

All expected worker nodes should eventually show `Ready`.

## 5. Troubleshooting

If nodes do not join the cluster, check:

- The node IAM role and its required permissions.
- Subnet routing, security groups, DNS, and connectivity to the EKS API endpoint.
- The node group status and associated Auto Scaling group in the AWS console.

See the [official EKS troubleshooting guide](https://docs.aws.amazon.com/eks/latest/userguide/troubleshooting.html#worker-node-fail).

## 6. Clean Up Learning Resources

Managed nodes are EC2 instances and incur charges while running. Delete the node group when you no longer need it:

```bash
eksctl delete nodegroup \
  --cluster "$CLUSTER_NAME" \
  --region "$AWS_REGION" \
  --name "$NODEGROUP_NAME"
```

Verify that the node group has been removed:

```bash
eksctl get nodegroup \
  --cluster "$CLUSTER_NAME" \
  --region "$AWS_REGION"
```

**Note:** This deletes the node group, not the EKS cluster itself. Review any workloads and storage before cleanup.

## References

- [Create a managed node group with EKS](https://docs.aws.amazon.com/eks/latest/userguide/create-managed-node-group.html)
- [Amazon EKS node IAM role](https://docs.aws.amazon.com/eks/latest/userguide/create-node-role.html)
- [Choose an EC2 instance type](https://docs.aws.amazon.com/eks/latest/userguide/choosing-instance-type.html)
