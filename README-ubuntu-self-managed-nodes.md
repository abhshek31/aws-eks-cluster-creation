# Create Self-Managed Ubuntu Linux Nodes on Amazon EKS

This guide creates a **self-managed Ubuntu node group** for an existing Amazon EKS cluster using `eksctl`.

> **Note:** This procedure is for clusters created with `eksctl`. Managed node groups are another option and simplify node lifecycle management. EC2 instances incur charges while running.

## 1. Prerequisites

Install and configure:

- [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)
- [kubectl](https://kubernetes.io/docs/tasks/tools/)
- [eksctl](https://eksctl.io/installation/)

The source guide requires `eksctl` version `0.215.0` or later.

Check versions and AWS identity:

```bash
aws --version
kubectl version --client
eksctl version
aws sts get-caller-identity
```

You need an existing EKS cluster created with `eksctl`, appropriate AWS permissions, and an EC2 SSH key pair if you want SSH access to the nodes.

## 2. Set Your Cluster Details

Replace the example values with your own:

```bash
export CLUSTER_NAME="my-cluster"
export AWS_REGION="us-east-1"
export NODEGROUP_NAME="ng-ubuntu"
export EC2_KEYPAIR="my-ec2-keypair-name"
```

Confirm the cluster is active:

```bash
aws eks describe-cluster \
  --name "$CLUSTER_NAME" \
  --region "$AWS_REGION" \
  --query 'cluster.status' \
  --output text
```

Expected output: `ACTIVE`.

## 3. Create the Node Group Configuration

Create `ubuntu.yaml`:

```bash
cat > ubuntu.yaml <<EOF
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: ${CLUSTER_NAME}
  region: ${AWS_REGION}

iam:
  withOIDC: true

nodeGroups:
  - name: ${NODEGROUP_NAME}
    instanceType: m5.large
    desiredCapacity: 3
    amiFamily: Ubuntu2204
    iam:
      attachPolicyARNs:
        - arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy
        - arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly
        - arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore
        - arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy
    ssh:
      allow: true
      publicKeyName: ${EC2_KEYPAIR}
EOF
```

Configuration notes:

- `Ubuntu2204` selects Ubuntu 22.04 nodes. To use Ubuntu Pro 22.04, change it to `UbuntuPro2204`.
- `m5.large` is an example x86 instance type. Choose an instance type compatible with your workload; use an Arm instance type for Arm nodes.
- The example attaches the policies listed in the source guide, including `AmazonEKS_CNI_Policy`. For a production setup, AWS recommends assigning CNI permissions to a dedicated role for the `aws-node` service account rather than the node role.
- SSH access is enabled in this example. Restrict inbound SSH access to trusted IP addresses and security groups. If you do not need SSH, remove the `ssh` section.

For Outposts, Wavelength, or Local Zone subnets, follow the [eksctl node group configuration guidance](https://eksctl.io/usage/nodegroups/#creating-a-nodegroup-from-a-config-file) and explicitly specify the appropriate subnets.

## 4. Create the Self-Managed Node Group

```bash
eksctl create nodegroup --config-file=ubuntu.yaml
```

Wait for the command to complete successfully.

## 5. Verify Nodes

Configure `kubectl` for the cluster:

```bash
aws eks update-kubeconfig \
  --name "$CLUSTER_NAME" \
  --region "$AWS_REGION"
```

Check node status:

```bash
kubectl get nodes -o wide
kubectl get nodes --watch
```

Wait for the expected nodes to show `Ready`. You can then deploy a [sample application](https://docs.aws.amazon.com/eks/latest/userguide/sample-deployment.html) to test the nodes.

## 6. Security Note

Consider restricting Pod access to the EC2 Instance Metadata Service (IMDS) when workloads use IAM roles for service accounts or EKS Pod Identity and do not otherwise require IMDS. Follow the [EKS best practices for restricting node instance-role access](https://aws.github.io/aws-eks-best-practices/security/docs/iam/#restrict-access-to-the-instance-profile-assigned-to-the-worker-node).

## 7. Clean Up

Self-managed nodes run on EC2 instances and incur charges. When finished, delete the node group:

```bash
eksctl delete nodegroup \
  --cluster "$CLUSTER_NAME" \
  --region "$AWS_REGION" \
  --name "$NODEGROUP_NAME"
```

Check that the node group has been removed:

```bash
eksctl get nodegroup \
  --cluster "$CLUSTER_NAME" \
  --region "$AWS_REGION"
```

## References

- [Create self-managed Ubuntu Linux nodes](https://docs.aws.amazon.com/eks/latest/userguide/launch-node-ubuntu.html)
- [eksctl node group configuration](https://eksctl.io/usage/nodegroups/#creating-a-nodegroup-from-a-config-file)
- [Amazon EKS node IAM role](https://docs.aws.amazon.com/eks/latest/userguide/create-node-role.html)
