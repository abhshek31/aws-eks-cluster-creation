# Amazon EKS Managed Node Group — User Guide

This guide creates an IAM node role and uses it to create a managed node group in an existing Amazon EKS cluster with `eksctl`.

## 1. Prerequisites

Install and configure:

- [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)
- [kubectl](https://kubernetes.io/docs/tasks/tools/)
- [eksctl](https://eksctl.io/installation/)

Check the tools and AWS identity:

```bash
aws --version
kubectl version --client
eksctl version
aws sts get-caller-identity
```

Use `eksctl` version `0.215.0` or later. You also need an existing EKS cluster in `ACTIVE` status and permissions to create IAM roles and EKS node groups.

## 2. Set Cluster Details

Replace the example values with your own:

```bash
export CLUSTER_NAME="my-cluster"
export AWS_REGION="us-east-1"
export NODEGROUP_NAME="my-mng"
export NODE_ROLE_NAME="EKSWorkerNodeRole"
```

Check that the cluster is active:

```bash
aws eks describe-cluster \
  --name "$CLUSTER_NAME" \
  --region "$AWS_REGION" \
  --query 'cluster.status' \
  --output text
```

Expected output: `ACTIVE`

Configure `kubectl`:

```bash
aws eks update-kubeconfig \
  --name "$CLUSTER_NAME" \
  --region "$AWS_REGION"
```

## 3. Create the EKS Node IAM Role

Worker nodes need an IAM role to communicate with EKS and pull images from ECR. This is separate from the IAM role used to create the EKS cluster.

Create the EC2 trust policy:

```bash
cat > node-trust-policy.json <<'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "ec2.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF
```

Create the role:

```bash
aws iam create-role \
  --role-name "$NODE_ROLE_NAME" \
  --assume-role-policy-document file://node-trust-policy.json
```

Attach the standard managed-node-group policies:

```bash
aws iam attach-role-policy \
  --role-name "$NODE_ROLE_NAME" \
  --policy-arn arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy

aws iam attach-role-policy \
  --role-name "$NODE_ROLE_NAME" \
  --policy-arn arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryPullOnly
```

If your nodes pull images from **Amazon ECR Public**, optionally attach this policy to avoid public-image pull throttling:

```bash
aws iam attach-role-policy \
  --role-name "$NODE_ROLE_NAME" \
  --policy-arn arn:aws:iam::aws:policy/AmazonElasticContainerRegistryPublicReadOnly
```

**Policy note:** `AmazonEKSWorkerNodeMinimalPolicy` is intended for **EKS Auto Mode** node roles, where it is used with `AmazonEC2ContainerRegistryPullOnly`. For this guide's standard managed node group, use `AmazonEKSWorkerNodePolicy` with `AmazonEC2ContainerRegistryPullOnly`; do not attach both worker-node policies by default. The VPC CNI also needs its own permissions, preferably through a separate IAM role associated with the `aws-node` service account. See the [AWS node IAM role guide](https://docs.aws.amazon.com/eks/latest/userguide/create-node-role.html).

Get the role ARN:

```bash
export NODE_ROLE_ARN=$(aws iam get-role \
  --role-name "$NODE_ROLE_NAME" \
  --query 'Role.Arn' \
  --output text)

echo "$NODE_ROLE_ARN"
```

**VPC CNI note:** For IPv4 clusters, `AmazonEKS_CNI_Policy` must be available to the VPC CNI. AWS recommends assigning it to a separate IAM role associated with the `aws-node` service account using IRSA or EKS Pod Identity. Attach it to the node role only if your CNI permissions are not provided separately. IPv6 clusters require an appropriate IPv6 CNI policy.

Official reference: [Amazon EKS node IAM role](https://docs.aws.amazon.com/eks/latest/userguide/create-node-role.html).

## 4. Create a Managed Node Group Using That Role

Create an `eksctl` configuration file that explicitly references the role ARN:

```bash
cat > eks-nodegroup.yaml <<EOF
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: ${CLUSTER_NAME}
  region: ${AWS_REGION}

managedNodeGroups:
  - name: ${NODEGROUP_NAME}
    instanceType: m5.large
    desiredCapacity: 3
    minSize: 2
    maxSize: 4
    iam:
      instanceRoleARN: ${NODE_ROLE_ARN}
EOF
```

Create the node group:

```bash
eksctl create nodegroup --config-file eks-nodegroup.yaml
```

The example starts with three `m5.large` nodes and allows the group size to range from two to four. Adjust the instance type and capacity to suit your workload and budget.

## 5. Verify the Nodes

Check the node group:

```bash
eksctl get nodegroup \
  --cluster "$CLUSTER_NAME" \
  --region "$AWS_REGION"
```

Check Kubernetes nodes:

```bash
kubectl get nodes -o wide
kubectl get nodes --watch
```

Wait until the expected nodes show `Ready`.

## 6. Troubleshooting

If nodes do not join the cluster, check:

- The node role trust policy and attached policies.
- Subnet routing, security groups, DNS, and connectivity to the EKS API endpoint.
- Node group status and EC2 Auto Scaling activity.

See the [official EKS troubleshooting guide](https://docs.aws.amazon.com/eks/latest/userguide/troubleshooting.html#worker-node-fail).

## 7. Clean Up

Managed node groups use EC2 instances and incur charges. Delete the node group when you finish learning:

```bash
eksctl delete nodegroup \
  --cluster "$CLUSTER_NAME" \
  --region "$AWS_REGION" \
  --name "$NODEGROUP_NAME"
```

This removes the node group, not the EKS cluster. Delete the IAM role separately only after confirming it is not used by another node group:

```bash
aws iam detach-role-policy \
  --role-name "$NODE_ROLE_NAME" \
  --policy-arn arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy

aws iam detach-role-policy \
  --role-name "$NODE_ROLE_NAME" \
  --policy-arn arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryPullOnly

# Run this too only if you attached the optional ECR Public policy:
aws iam detach-role-policy \
  --role-name "$NODE_ROLE_NAME" \
  --policy-arn arn:aws:iam::aws:policy/AmazonElasticContainerRegistryPublicReadOnly

aws iam delete-role --role-name "$NODE_ROLE_NAME"
```

## References

- [Create a managed node group](https://docs.aws.amazon.com/eks/latest/userguide/create-managed-node-group.html)
- [Amazon EKS node IAM role](https://docs.aws.amazon.com/eks/latest/userguide/create-node-role.html)
