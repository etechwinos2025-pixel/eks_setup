# GitHub Actions OIDC Setup for AWS

This guide explains how to configure GitHub Actions to authenticate with AWS using OpenID Connect (OIDC) without storing long-lived credentials.

## Why OIDC?

- ✅ **No long-lived credentials** stored in GitHub Secrets
- ✅ **Automatic credential rotation** by AWS STS
- ✅ **Fine-grained permissions** via IAM policies
- ✅ **Audit trail** of which GitHub runs accessed AWS
- ✅ **More secure** than access keys

## Prerequisites

- AWS Account with IAM permissions
- GitHub repository (public or private)
- AWS CLI v2 configured

## Step 1: Create an IAM OIDC Provider

Run this command to create the GitHub OIDC provider in your AWS account:

```bash
aws iam create-openid-connect-provider \
  --url https://token.actions.githubusercontent.com \
  --client-id-list sts.amazonaws.com \
  --thumbprint-list 6938fd4d98bab03faadb97b34396831e3780aea1 \
  --region us-east-1
```

**Output example:**
```
{
    "OpenIDConnectProviderArn": "arn:aws:iam::123456789012:oidc-provider/token.actions.githubusercontent.com"
}
```

Save the ARN - you'll need it in the next step.

## Step 2: Create an IAM Role for GitHub Actions

Create a role that GitHub Actions can assume. Save this as `trust-policy.json`:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::YOUR_ACCOUNT_ID:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
        },
        "StringLike": {
          "token.actions.githubusercontent.com:sub": "repo:etechwinos2025-pixel/eks_setup:*"
        }
      }
    }
  ]
}
```

Replace `YOUR_ACCOUNT_ID` with your actual AWS account ID.

Create the role:

```bash
aws iam create-role \
  --role-name GitHubActionsEKSRole \
  --assume-role-policy-document file://trust-policy.json \
  --region us-east-1
```

**Output:**
```
{
    "Role": {
        "Arn": "arn:aws:iam::123456789012:role/GitHubActionsEKSRole",
        ...
    }
}
```

Save the Role ARN.

## Step 3: Attach IAM Policies

Attach the necessary policies to allow Terraform to create EKS resources:

```bash
# For EKS and VPC permissions
aws iam attach-role-policy \
  --role-name GitHubActionsEKSRole \
  --policy-arn arn:aws:iam::aws:policy/AdministratorAccess \
  --region us-east-1
```

**⚠️ Note:** `AdministratorAccess` is permissive for demo purposes. For production, create a custom policy with minimal permissions:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ec2:*",
        "eks:*",
        "iam:*",
        "logs:*",
        "autoscaling:*",
        "elasticloadbalancing:*"
      ],
      "Resource": "*"
    }
  ]
}
```

## Step 4: Add GitHub Secret

Add the IAM Role ARN to your GitHub repository secrets:

1. Go to **Settings** → **Secrets and variables** → **Actions**
2. Click **New repository secret**
3. Name: `AWS_ROLE_ARN`
4. Value: `arn:aws:iam::123456789012:role/GitHubActionsEKSRole`
5. Click **Add secret**

## Step 5: Verify Workflow

The workflow file is already configured to use OIDC. Just push a change to trigger the pipeline:

```bash
git add .
git commit -m "test: Trigger terraform workflow"
git push origin main
```

Watch the pipeline in **Actions** tab:
1. **Validate** - Checks Terraform syntax
2. **Plan** - Shows what will be created (on PRs only)
3. **Apply** - Creates infrastructure (on main push only)

## Workflow Triggers

| Event | Behavior |
|-------|----------|
| **Pull Request** | Validates + Plans (no apply) |
| **Push to main** | Validates + Plans + Applies |

## Monitoring the Deployment

1. Go to **Actions** → **Terraform Plan & Apply**
2. Click the workflow run
3. View logs in real-time
4. Check **Terraform Apply** for cluster outputs

## Troubleshooting

### Error: "Failed to assume role"
- Verify Role ARN in GitHub Secrets is correct
- Check trust policy includes your GitHub repo
- Ensure OIDC provider is created

### Error: "Access Denied"
- Verify IAM policy is attached to the role
- Check CloudTrail for denied actions
- May need `AdministratorAccess` or custom policy

### Workflow stuck on Plan
- Check if previous apply is still running
- Look at AWS CloudFormation events in console
- May need to destroy and retry

## Cleanup

To destroy the infrastructure via GitHub Actions, use a manual workflow or comment trigger:

```bash
# Manually trigger destroy (requires separate workflow)
# Or destroy locally:
terraform destroy
```

## Security Best Practices

✅ Use `StringLike` condition for specific repos
✅ Rotate GitHub OIDC thumbprint annually
✅ Use custom IAM policies with least privilege
✅ Enable CloudTrail logging
✅ Review Actions logs regularly

## Additional Resources

- [AWS OIDC Documentation](https://docs.aws.amazon.com/iam/latest/userguide/id_roles_providers_create_oidc.html)
- [GitHub OIDC Guide](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/about-security-hardening-with-openid-connect)
- [Terraform AWS Provider](https://registry.terraform.io/providers/hashicorp/aws/latest/docs)

---

Once configured, the workflow will automatically deploy your EKS cluster whenever you push to main!
