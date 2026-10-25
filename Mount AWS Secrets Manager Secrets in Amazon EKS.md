# Mount AWS Secrets Manager Secrets in Amazon EKS Using EKS Pod Identity

The solution retrieves a secret from **AWS Secrets Manager** and mounts it as a file inside an NGINX pod using the **AWS Secrets Store CSI Driver provider** and **EKS Pod Identity**.

## How it works

1. Store a secret in AWS Secrets Manager.
2. Install the AWS Secrets Store CSI Driver provider EKS add-on and the EKS Pod Identity Agent.
3. Create an IAM role with permission to read secrets and associate it with the Kubernetes service account.
4. Define the secret to mount in a `SecretProviderClass`.
5. Deploy NGINX. The CSI volume mounts the secret file at `/mnt/secrets-store/`.

> **Security note:** The example below creates a test secret. Do not put real credentials in source control or print production secret values in logs. For production, scope IAM permissions to only the required secrets.

## Prerequisites

- An Amazon EKS cluster and access to its AWS account.
- AWS CLI v2, `kubectl`, and `eksctl` installed and configured.
- IAM permissions to manage EKS add-ons, Pod Identity associations, IAM roles/policies, and Secrets Manager secrets.
- Kubernetes access to the target EKS cluster.

Set your environment variables. Replace the values with your own:

```bash
export AWS_REGION="us-east-1"
export CLUSTER_NAME="my-test-cluster"
export SECRET_NAME="addon_secret"
export IAM_ROLE_NAME="nginx-deployment-role"

aws configure set default.region "$AWS_REGION"
aws sts get-caller-identity
aws eks update-kubeconfig --region "$AWS_REGION" --name "$CLUSTER_NAME"
kubectl get nodes
```

If you do not already have a cluster, the AWS article uses this command to create a sample cluster (this can take around 15 minutes):

```bash
eksctl create cluster "$CLUSTER_NAME" --region "$AWS_REGION"
```

## 1. Create a test secret

```bash
aws secretsmanager create-secret \
  --region "$AWS_REGION" \
  --name "$SECRET_NAME" \
  --secret-string "super secret!"
```

If a secret with this name already exists, use a different name or update the existing secret instead of creating it again.

## 2. Install the EKS add-ons

Install the AWS provider for the Secrets Store CSI Driver:

```bash
eksctl create addon \
  --cluster "$CLUSTER_NAME" \
  --region "$AWS_REGION" \
  --name aws-secrets-store-csi-driver-provider
```

Install the EKS Pod Identity Agent:

```bash
eksctl create addon \
  --cluster "$CLUSTER_NAME" \
  --region "$AWS_REGION" \
  --name eks-pod-identity-agent
```

Check the add-ons:

```bash
aws eks list-addons --cluster-name "$CLUSTER_NAME" --region "$AWS_REGION"
kubectl get pods -n kube-system
```

The CSI driver itself must also be installed in the cluster. Check the add-on and CSI driver installation status if the workload reports that the CSI driver or volume is unavailable.

## 2A. Enable automatic secret rotation

The Secrets Store CSI Driver can periodically refresh mounted secret files after the value in AWS Secrets Manager changes. Configure the **`secrets-store-csi-driver` EKS add-on** with secret rotation enabled.

Create **`configuration-value.json`**:

```json
{
  "secrets-store-csi-driver": {
    "enableSecretRotation": true,
    "rotationPollInterval": "1m",
    "syncSecret": {
      "enabled": true
    }
  }
}
```

Apply the configuration to the add-on:

```bash
aws eks update-addon \\
  --cluster-name "$CLUSTER_NAME" \\
  --region "$AWS_REGION" \\
  --addon-name secrets-store-csi-driver \\
  --configuration-values file://configuration-value.json \\
  --resolve-conflicts PRESERVE
```

Check the add-on status:

```bash
aws eks describe-addon \\
  --cluster-name "$CLUSTER_NAME" \\
  --region "$AWS_REGION" \\
  --addon-name secrets-store-csi-driver \\
  --query 'addon.{status:status,configuration:configurationValues}'
```

**How rotation works**

1. Update the secret value in AWS Secrets Manager.
2. The CSI driver's rotation reconciler polls according to `rotationPollInterval` (here, every `1m`) and refreshes the mounted file when a newer value is available.
3. The application must read the updated file again or reload its configuration. Updating a mounted file does not automatically make every application reload the secret.

**Important details**

- The `secrets-store-csi-driver` add-on must already be installed before you run `aws eks update-addon`. If it is not installed, create it with the same configuration using `aws eks create-addon` and `--configuration-values file://configuration-value.json`.
- Rotation updates the mounted files. It does not automatically restart pods.
- `syncSecret.enabled: true` enables synchronization to a native Kubernetes Secret only when the `SecretProviderClass` also defines `secretObjects`. The `SecretProviderClass` in this guide does not define `secretObjects`, so this configuration alone will not create a Kubernetes Secret. Mounting the secret as a file continues to work without `secretObjects`.
- Rotation polling can increase requests to the secret provider. Choose an interval that balances freshness and operational overhead, and verify the configuration options supported by the EKS add-on version in use.

## 3. Create an IAM role for EKS Pod Identity

Create a trust policy file named `pod-identity-trust-policy.json`:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "pods.eks.amazonaws.com"
      },
      "Action": [
        "sts:AssumeRole",
        "sts:TagSession"
      ]
    }
  ]
}
```

Create the role:

```bash
aws iam create-role \
  --role-name "$IAM_ROLE_NAME" \
  --assume-role-policy-document file://pod-identity-trust-policy.json
```

Attach the AWS-managed read-only policy used in the article:

```bash
aws iam attach-role-policy \
  --role-name "$IAM_ROLE_NAME" \
  --policy-arn arn:aws:iam::aws:policy/AWSSecretsManagerClientReadOnlyAccess
```

Get the role ARN:

```bash
export ROLE_ARN="$(aws iam get-role \
  --role-name "$IAM_ROLE_NAME" \
  --query 'Role.Arn' \
  --output text)"

echo "$ROLE_ARN"
```

**Least-privilege recommendation:** `AWSSecretsManagerClientReadOnlyAccess` can grant read access to secrets across the account. For production, consider a custom IAM policy restricted to the exact secret ARN(s) required by this workload.

## 4. Associate the IAM role with the Kubernetes service account

The association must use the same namespace and service account name as the deployment. The manifest below uses the `default` namespace and service account `nginx-pod-identity-deployment-sa`.

```bash
eksctl create podidentityassociation \
  --cluster "$CLUSTER_NAME" \
  --region "$AWS_REGION" \
  --namespace default \
  --service-account-name nginx-pod-identity-deployment-sa \
  --role-arn "$ROLE_ARN" \
  --create-service-account
```

## 5. Create the SecretProviderClass

Save the following as **`secretproviderclass.yaml`**. The `metadata.name` must match the `secretProviderClass` value in the deployment, and `objectName` must match the secret name in Secrets Manager.

```yaml
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: nginx-pod-identity-deployment-aws-secrets
spec:
  provider: aws
  parameters:
    objects: |
      - objectName: "addon_secret"
        objectType: "secretsmanager"
    usePodIdentity: "true"
```

Apply it:

```bash
kubectl apply -f secretproviderclass.yaml
kubectl get secretproviderclass
```

If you changed `SECRET_NAME` earlier, update `objectName` in this YAML to the same value.

## 6. Deploy the Service and NGINX Deployment

Save the following as **`deployment.yaml`**. This is the deployment used in this guide.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-pod-identity-deployment
  labels:
    app: nginx-pod-identity
spec:
  selector:
    app: nginx-pod-identity
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-pod-identity-deployment
  labels:
    app: nginx-pod-identity
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx-pod-identity
  template:
    metadata:
      labels:
        app: nginx-pod-identity
    spec:
      serviceAccountName: nginx-pod-identity-deployment-sa
      volumes:
        - name: secrets-store-inline
          csi:
            driver: secrets-store.csi.k8s.io
            readOnly: true
            volumeAttributes:
              secretProviderClass: "nginx-pod-identity-deployment-aws-secrets"
      containers:
        - name: nginx-pod-identity-deployment
          image: nginx
          ports:
            - containerPort: 80
          volumeMounts:
            - name: secrets-store-inline
              mountPath: "/mnt/secrets-store"
              readOnly: true
```

Apply the manifest and wait for the pods to become ready:

```bash
kubectl apply -f deployment.yaml
kubectl rollout status deployment/nginx-pod-identity-deployment
kubectl get deployment,service,pods
```

The Service exposes port 80 inside the cluster. To test NGINX locally from your workstation, use port-forwarding:

```bash
kubectl port-forward service/nginx-pod-identity-deployment 8080:80
```

Then open `http://localhost:8080` in a browser.

## 7. Verify the mounted secret

List the running pods:

```bash
kubectl get pods -l app=nginx-pod-identity
```

Read the secret file from one pod:

```bash
POD_NAME="$(kubectl get pods \
  -l app=nginx-pod-identity \
  -o jsonpath='{.items[0].metadata.name}')"

kubectl exec "$POD_NAME" -- ls -l /mnt/secrets-store/
kubectl exec "$POD_NAME" -- cat /mnt/secrets-store/addon_secret
```

For the test secret created above, the output should be:

```text
super secret!
```

The secret is mounted as a **file**, not automatically as an environment variable or a native Kubernetes `Secret`.

## 8. Troubleshooting

Check the deployment and pod events:

```bash
kubectl describe deployment nginx-pod-identity-deployment
kubectl get pods -l app=nginx-pod-identity
kubectl describe pod "$POD_NAME"
kubectl get events --sort-by=.lastTimestamp
```

Check the relevant add-on pods:

```bash
kubectl get pods -n kube-system
```

Common checks:

- **Secret file is missing:** Confirm the CSI driver and AWS provider add-on are installed and healthy.
- **Access denied:** Confirm the Pod Identity association points to the correct IAM role, namespace, and service account, and that the role can read the target secret.
- **SecretProviderClass not found:** Ensure `secretproviderclass.yaml` was applied in the same namespace as the pod.
- **Name mismatch:** Ensure the deployment's `secretProviderClass` exactly matches `metadata.name`, and `objectName` matches the Secrets Manager secret name.
- **Pod Identity unavailable:** Confirm the EKS Pod Identity Agent add-on is installed and running.

## 9. Clean up

Delete the Kubernetes resources:

```bash
kubectl delete -f deployment.yaml
kubectl delete -f secretproviderclass.yaml
```

Delete the Pod Identity association using the AWS console or the matching `eksctl delete podidentityassociation` command for your installed `eksctl` version. Confirm the association is no longer needed before removing it.

Detach the managed policy and delete the IAM role:

```bash
aws iam detach-role-policy \
  --role-name "$IAM_ROLE_NAME" \
  --policy-arn arn:aws:iam::aws:policy/AWSSecretsManagerClientReadOnlyAccess

aws iam delete-role --role-name "$IAM_ROLE_NAME"
```

Delete the test secret (this bypasses the recovery window):

```bash
aws secretsmanager delete-secret \
  --region "$AWS_REGION" \
  --secret-id "$SECRET_NAME" \
  --force-delete-without-recovery
```

If you created a dedicated test EKS cluster just for this guide, delete it after removing any resources you want to retain:

```bash
eksctl delete cluster "$CLUSTER_NAME" --region "$AWS_REGION"
```

Do not delete a shared or production cluster as part of cleanup.

## References

- [AWS Security Blog: Secrets Store CSI Driver provider EKS add-on with Secrets Manager](https://aws.amazon.com/blogs/security/how-to-use-the-secrets-store-csi-driver-provider-amazon-eks-add-on-with-secrets-manager/)
- [AWS Secrets Store CSI Driver provider for AWS (GitHub)](https://github.com/aws/secrets-store-csi-driver-provider-aws)
- [Amazon EKS Pod Identity](https://docs.aws.amazon.com/eks/latest/userguide/pod-identities.html)
- [AWS Secrets Manager](https://docs.aws.amazon.com/secretsmanager/latest/userguide/intro.html)
