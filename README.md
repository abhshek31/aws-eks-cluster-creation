# aws-eks-cluster-creation
# EKS Learning -- User Guide

A step-by-step guide to creating an Amazon Elastic Kubernetes Service
(Amazon EKS) cluster in **AWS Region `us-east-1`** using the AWS
Management Console, with supporting AWS CLI commands for prerequisites,
networking, and IAM.

> **Learning note:** Creating the EKS control plane does not
> automatically give you worker nodes in this workflow. After the
> cluster becomes `ACTIVE`, you can add a managed node group or
> configure another compute option before deploying workloads.

## Table of Contents

-   [Overview](#overview)
-   [Prerequisites](#prerequisites)
-   [Step 1: Configure the AWS CLI](#step-1-configure-the-aws-cli)
-   [Step 2: Verify Your AWS Identity](#step-2-verify-your-aws-identity)
-   [Step 3: Create the VPC and
    Subnets](#step-3-create-the-vpc-and-subnets)
-   [Step 4: Create the EKS Cluster IAM
    Role](#step-4-create-the-eks-cluster-iam-role)
-   [Step 5: Create the EKS Cluster](#step-5-create-the-eks-cluster)
-   [Step 6: Configure kubectl](#step-6-configure-kubectl)
-   [Step 7: Verify the Cluster](#step-7-verify-the-cluster)
-   [Troubleshooting](#troubleshooting)
-   [Cost and Cleanup](#cost-and-cleanup)
-   [Useful References](#useful-references)

## Overview

This guide walks through the following resources:

1.  AWS CLI and `kubectl` on your workstation.
2.  An Amazon VPC with public and private subnets.
3.  An IAM role trusted by the Amazon EKS service and attached to
    `AmazonEKSClusterPolicy`.
4.  An EKS cluster created with **Custom configuration**, with **EKS
    Auto Mode disabled**.
5.  A local `kubectl` context for communicating with the cluster.

### Values used in this guide

  Setting                    Value
  -------------------------- -------------------------------------------
  AWS Region                 `us-east-1`
  CloudFormation stack       `my-eks-vpc-stack`
  Cluster IAM role           `myAmazonEKSClusterRole`
  EKS cluster example name   `my-cluster`
  VPC template               Amazon EKS public/private subnet template

Change the region and names consistently if you want to use a different
environment.

## Prerequisites

Before starting, install and configure the following tools and
resources.

### 1. AWS CLI v2

Install AWS CLI version 2 by following the [official installation
guide](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html).

Verify the installation:

``` bash
aws --version
```

The output should identify AWS CLI version `2.x`.

Configure credentials using your organization's approved authentication
method. For a basic local learning setup, you can use:

``` bash
aws configure
```

Provide the access key, secret key, default region (`us-east-1`), and
output format when prompted. Prefer temporary credentials or IAM
Identity Center for regular use where possible; do not commit
credentials to this repository.

### 2. kubectl

Install `kubectl` by following the [official Kubernetes tools
guide](https://kubernetes.io/docs/tasks/tools/).

Verify the installation:

``` bash
kubectl version --client
```

The `kubectl` version should be compatible with the Kubernetes version
used by your EKS cluster. See the [EKS kubectl
guidance](https://docs.aws.amazon.com/eks/latest/userguide/install-kubectl.html).

### 3. Required AWS IAM permissions

The IAM identity you use must have sufficient permissions to create and
manage the required resources, including:

-   Amazon EKS clusters and related operations.
-   IAM roles and role-policy attachments.
-   AWS CloudFormation stacks.
-   VPC networking resources and related EC2 operations.

Use the same AWS identity throughout this guide to avoid ownership and
access confusion. In production, follow least-privilege practices and
have an administrator grant only the permissions required for your task.

## Step 1: Configure the AWS CLI

Set the region for this terminal session:

``` bash
export AWS_REGION=us-east-1
export AWS_DEFAULT_REGION="$AWS_REGION"
```

Check the CLI version:

``` bash
aws --version
```

If you use a named AWS CLI profile, add `--profile YOUR_PROFILE` to AWS
CLI commands, or set:

``` bash
export AWS_PROFILE=YOUR_PROFILE
```

Replace `YOUR_PROFILE` with the profile name configured on your machine.

## Step 2: Verify Your AWS Identity

Run:

``` bash
aws sts get-caller-identity
```

Example output:

``` json
{
  "UserId": "EXAMPLEUSERID",
  "Account": "123456789012",
  "Arn": "arn:aws:iam::123456789012:user/example-user"
}
```

The values above are illustrative. Confirm that the returned AWS account
and identity are the ones where you intend to create the EKS resources.

## Step 3: Create the VPC and Subnets

Amazon EKS needs a VPC and appropriately configured subnets. This
example uses the AWS-provided CloudFormation template for a VPC with
public and private subnets.

Create the stack:

``` bash
aws cloudformation create-stack \
  --region "$AWS_REGION" \
  --stack-name my-eks-vpc-stack \
  --template-url https://s3.us-west-2.amazonaws.com/amazon-eks/cloudformation/2020-10-29/amazon-eks-vpc-private-subnets.yaml
```

Check the stack status:

``` bash
aws cloudformation describe-stacks \
  --region "$AWS_REGION" \
  --stack-name my-eks-vpc-stack \
  --query 'Stacks[0].StackStatus' \
  --output text
```

Wait for the stack to finish:

``` bash
aws cloudformation wait stack-create-complete \
  --region "$AWS_REGION" \
  --stack-name my-eks-vpc-stack
```

List the stack outputs:

``` bash
aws cloudformation describe-stacks \
  --region "$AWS_REGION" \
  --stack-name my-eks-vpc-stack \
  --query 'Stacks[0].Outputs' \
  --output table
```

**Important:** If stack creation fails, inspect the CloudFormation
events before proceeding. The template is hosted in `us-west-2`, but the
stack is created in `us-east-1`; this is intentional because the
template URL points to the S3 bucket hosting the template.

## Step 4: Create the EKS Cluster IAM Role

The EKS cluster needs an IAM role that the Amazon EKS service can
assume.

### 4.1 Create the trust policy file

In your repository or working directory, create a file named
`eks-cluster-role-trust-policy.json` with the following content:

``` json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "eks.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

### 4.2 Create the IAM role

``` bash
aws iam create-role \
  --role-name myAmazonEKSClusterRole \
  --assume-role-policy-document file://eks-cluster-role-trust-policy.json
```

If the role already exists, do not blindly rerun the create command.
Inspect the existing role and confirm its trust policy is correct.

### 4.3 Attach the required EKS cluster policy

``` bash
aws iam attach-role-policy \
  --policy-arn arn:aws:iam::aws:policy/AmazonEKSClusterPolicy \
  --role-name myAmazonEKSClusterRole
```

Verify the attached policy:

``` bash
aws iam list-attached-role-policies \
  --role-name myAmazonEKSClusterRole \
  --output table
```

### 4.4 Retrieve the role ARN

You will select this role in the EKS console. To display its ARN:

``` bash
aws iam get-role \
  --role-name myAmazonEKSClusterRole \
  --query 'Role.Arn' \
  --output text
```

Keep this role ARN available. Do not confuse the **cluster IAM role**
with the separate IAM role used by worker nodes or workloads.

## Step 5: Create the EKS Cluster

You can create the cluster through the AWS Management Console.

1.  Open the [Amazon EKS Clusters
    console](https://console.aws.amazon.com/eks/home#/clusters).
2.  Confirm that the selected AWS Region is **US East (N. Virginia) --
    `us-east-1`**.
3.  Choose **Create cluster**. If the option is not visible, select
    **Clusters** from the left navigation.
4.  On the cluster configuration page:
    -   Select **Custom configuration**.
    -   Disable **Use EKS Auto Mode** for this learning path.
    -   Enter a cluster name, for example `my-cluster`.
    -   For **Cluster service role**, select `myAmazonEKSClusterRole`.
    -   Leave other settings at their defaults unless your learning
        exercise requires changes, then choose **Next**.
5.  On the networking page:
    -   Select the VPC created by `my-eks-vpc-stack` (its name should
        resemble `my-eks-vpc-stack-VPC`).
    -   Select the subnets created by the same stack.
    -   Select the stack's control-plane security group under
        **Additional security groups**, if the console presents it as
        required for this workflow.
    -   Review the settings and choose **Next**.
6.  On **Configure observability**, choose **Next** or configure the
    options you want to learn.
7.  On **Select add-ons**, review the available add-ons and choose
    **Next**. Configure selected add-on settings if prompted.
8.  Review the configuration and choose **Create**.
9.  Wait for the cluster status to become **Active** before continuing.

Cluster creation can take several minutes. If the cluster fails to
create, review the status and events in the EKS console before retrying.


## Step 6: Configure kubectl

After the cluster status becomes `ACTIVE`, configure your local
Kubernetes client.

Set the cluster name if you used a different name:

``` bash
export CLUSTER_NAME=my-cluster
```

Update your kubeconfig:

``` bash
aws eks update-kubeconfig \
  --region "$AWS_REGION" \
  --name "$CLUSTER_NAME"
```

This command updates the default kubeconfig file (normally
`~/.kube/config`) and adds or updates a context for the cluster.

Confirm the current context:

``` bash
kubectl config current-context
```

List configured contexts:

``` bash
kubectl config get-contexts
```

If you use a named AWS profile, use the same profile for
`aws eks update-kubeconfig` that you intend to use for cluster
authentication.

## Step 7: Verify the Cluster

Check the Kubernetes API connection:

``` bash
kubectl cluster-info
```

Check the cluster nodes:

``` bash
kubectl get nodes -o wide
```

**Expected learning outcome:** the cluster control plane may be active
even if `kubectl get nodes` returns `No resources found`. This guide
creates the cluster control plane but does not add a worker node group.
To schedule application pods, add an EKS managed node group or another
supported compute option and ensure the appropriate node IAM role and
networking are configured.

Check system pods:

``` bash
kubectl get pods -n kube-system
```

You should see Kubernetes system components and any system pods that are
scheduled for the configured compute. The exact output depends on the
cluster configuration and add-ons.

## Troubleshooting

### `aws eks update-kubeconfig` fails

-   Confirm AWS CLI v2 is installed: `aws --version`.
-   Confirm the cluster is `ACTIVE`.
-   Confirm the region and cluster name are correct.
-   Confirm your current identity has permission to describe the
    cluster.
-   Retry with the same AWS CLI profile used for this environment.

### The cluster is not becoming `ACTIVE`

-   Inspect the cluster status and issue in the EKS console.
-   Check that the cluster role exists and has `AmazonEKSClusterPolicy`
    attached.
-   Confirm the VPC and subnets were created successfully and meet EKS
    networking requirements.
-   Review CloudFormation events for failures in the VPC stack.

### `kubectl` reports `Unauthorized` or `Forbidden`

-   Check the identity returned by `aws sts get-caller-identity`.
-   Confirm that the identity is authorized to access the cluster
    through the EKS access configuration.
-   Remember that AWS IAM permissions to create the cluster do not
    automatically grant every IAM user or role Kubernetes administrator
    permissions. Configure EKS access entries or the applicable access
    mechanism for additional users and roles.

### `kubectl get nodes` shows no nodes

This is expected until compute is added. Create a managed node group or
configure another supported compute option before deploying workloads.

## Cost and Cleanup

**AWS resources can incur charges.** EKS cluster usage, EC2 instances,
EBS volumes, NAT gateways, load balancers, and other supporting
resources may incur costs. Review current AWS pricing before creating
resources, and remove learning resources when you no longer need them.

Before deleting resources:

1.  Delete workloads and services that provision AWS load balancers.
2.  Delete any managed node groups or other compute resources you
    created.
3.  Delete the EKS cluster.
4.  Delete the VPC CloudFormation stack after the resources using its
    networking have been removed.
5.  Delete the cluster IAM role only if it is no longer used by any
    cluster or other workflow.

For example, after you have confirmed the cluster is no longer needed
and removed dependent resources, delete the cluster:

``` bash
aws eks delete-cluster \
  --region "$AWS_REGION" \
  --name "$CLUSTER_NAME"
```

Wait until the cluster has been deleted before removing networking
resources. Then delete the VPC stack:

``` bash
aws cloudformation delete-stack \
  --region "$AWS_REGION" \
  --stack-name my-eks-vpc-stack
```

Check the stack deletion status:

``` bash
aws cloudformation describe-stacks \
  --region "$AWS_REGION" \
  --stack-name my-eks-vpc-stack
```

After successful deletion, CloudFormation no longer returns the stack.
Do not delete a stack if it contains resources you still need.

If the IAM role is dedicated to this learning cluster and is no longer
used, detach the policy and delete the role:

``` bash
aws iam detach-role-policy \
  --policy-arn arn:aws:iam::aws:policy/AmazonEKSClusterPolicy \
  --role-name myAmazonEKSClusterRole

aws iam delete-role \
  --role-name myAmazonEKSClusterRole
```

## Useful References

-   [Amazon EKS User
    Guide](https://docs.aws.amazon.com/eks/latest/userguide/what-is-eks.html)
-   [Create an Amazon EKS
    cluster](https://docs.aws.amazon.com/eks/latest/userguide/create-cluster.html)
-   [Amazon EKS VPC requirements and
    considerations](https://docs.aws.amazon.com/eks/latest/userguide/network-reqs.html)
-   [Install or update AWS
    CLI](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)
-   [Install `kubectl` for
    EKS](https://docs.aws.amazon.com/eks/latest/userguide/install-kubectl.html)
-   [Amazon EKS pricing](https://aws.amazon.com/eks/pricing/)

------------------------------------------------------------------------

Happy learning! Keep experimenting in a dedicated AWS learning account
and clean up unused resources to avoid unexpected charges.
