# EKS: Read AWS Secrets Manager Secrets Using Secrets Store CSI Driver (IRSA)

This document provides **end-to-end working steps** to configure Amazon EKS to read secrets securely from **AWS Secrets Manager** using:

- Secrets Store CSI Driver
- AWS Secrets Store CSI Driver Provider
- IRSA (IAM Role for Service Account)

---

## Environment Details

| Item | Value |
|-----|------|
| AWS Account ID | 865783518572 |
| Region | us-east-1 |
| EKS Cluster | ai-eks |
| Namespace | apipark |
| ServiceAccount | apipark-sa |
| Secrets Manager Secret | apipark/secrets |
| IAM Role | eks-apipark-irsa-role |
| IAM Policy | ReadAllSecretsManager |

---

## 1. Prerequisites

- awscli, kubectl, helm, eksctl installed
- Valid access to the AWS account
- kubeconfig updated

```bash
aws eks update-kubeconfig --name ai-eks --region us-east-1
kubectl get nodes
```

---

## 2. Install Secrets Store CSI Driver (Helm)

```bash
helm repo add secrets-store-csi-driver https://kubernetes-sigs.github.io/secrets-store-csi-driver/charts
helm repo update

helm upgrade --install csi-secrets-store   secrets-store-csi-driver/secrets-store-csi-driver   -n kube-system   --create-namespace   --set syncSecret.enabled=true   --set enableSecretRotation=true
```

Verify:

```bash
kubectl get csidriver
kubectl get pods -n kube-system | grep secrets
```

---

## 3. Install AWS Provider (EKS Add-on)

```bash
eksctl create addon   --cluster ai-eks   --name aws-secrets-store-csi-driver-provider   --region us-east-1   --force
```

---

## 4. Enable OIDC Provider

```bash
eksctl utils associate-iam-oidc-provider   --cluster ai-eks   --region us-east-1   --approve
```

Fetch OIDC Provider:

```bash
OIDC_ISSUER=$(aws eks describe-cluster   --name ai-eks   --region us-east-1   --query "cluster.identity.oidc.issuer"   --output text)

OIDC_PROVIDER=$(echo "$OIDC_ISSUER" | sed 's#^https://##')
echo $OIDC_PROVIDER
```

---

## 5. IAM Policy: Read All Secrets

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "secretsmanager:ListSecrets",
        "secretsmanager:DescribeSecret"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "secretsmanager:GetSecretValue"
      ],
      "Resource": "arn:aws:secretsmanager:*:865783518572:secret:*"
    }
  ]
}
```

Create policy:

```bash
aws iam create-policy   --policy-name ReadAllSecretsManager   --policy-document file://sm-read-all.json
```

---

## 6. Create IRSA Role

Trust policy (replace OIDC_PROVIDER):

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::865783518572:oidc-provider/OIDC_PROVIDER"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "OIDC_PROVIDER:sub": "system:serviceaccount:apipark:apipark-sa"
        }
      }
    }
  ]
}
```

Create role:

```bash
aws iam create-role   --role-name eks-apipark-irsa-role   --assume-role-policy-document file://trust.json
```

Attach policy:

```bash
aws iam attach-role-policy   --role-name eks-apipark-irsa-role   --policy-arn arn:aws:iam::865783518572:policy/ReadAllSecretsManager
```

---

## 7. Create Namespace + ServiceAccount

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: apipark
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: apipark-sa
  namespace: apipark
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::865783518572:role/eks-apipark-irsa-role
```

Apply:

```bash
kubectl apply -f sa-apipark.yaml
```

---

## 8. SecretProviderClass

```yaml
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: apipark-secrets-spc
  namespace: apipark
spec:
  provider: aws
  parameters:
    objects: |
      - objectName: "apipark/secrets"
        objectType: "secretsmanager"
        objectAlias: "apipark-secrets.json"
  secretObjects:
    - secretName: apipark-secrets-k8s
      type: Opaque
      data:
        - objectName: "apipark/secrets"
          key: "apipark-secrets.json"
```

---

## 9. Test Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: apipark-test
  namespace: apipark
spec:
  replicas: 1
  selector:
    matchLabels:
      app: apipark-test
  template:
    metadata:
      labels:
        app: apipark-test
    spec:
      serviceAccountName: apipark-sa
      containers:
        - name: test
          image: public.ecr.aws/docker/library/busybox
          command: ["sh", "-c", "sleep 3600"]
          volumeMounts:
            - name: secrets
              mountPath: /mnt/secrets
              readOnly: true
      volumes:
        - name: secrets
          csi:
            driver: secrets-store.csi.k8s.io
            readOnly: true
            volumeAttributes:
              secretProviderClass: apipark-secrets-spc
```

---

## 10. Validation

```bash
kubectl exec -it deploy/apipark-test -n apipark -- ls /mnt/secrets
kubectl exec -it deploy/apipark-test -n apipark -- cat /mnt/secrets/apipark-secrets.json
```

---

## Done 🎉

Your EKS cluster now securely reads secrets from AWS Secrets Manager using IRSA and CSI Driver.
