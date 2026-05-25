# EKS Setup - Simple Kubernetes Cluster on AWS

A minimal, production-ready Terraform configuration to deploy an Amazon EKS (Elastic Kubernetes Service) cluster with managed node groups in **us-east-1**.

## Features

- ✅ **Simple & Minimal**: Core EKS infrastructure without add-ons
- ✅ **Managed Node Groups**: Two auto-scaling node groups with t3.small instances
- ✅ **VPC Setup**: Custom VPC with 3 availability zones, public & private subnets
- ✅ **IAM Integration**: Cluster creator has admin access
- ✅ **Production Ready**: Security best practices, proper tagging, state management
- ✅ **Easy Kubectl Access**: Auto-generated kubectl configuration command

## Prerequisites

- **Terraform** >= 1.3
- **AWS CLI** v2
- **AWS Account** with appropriate permissions
- **kubectl** (optional, for cluster interaction)

## Quick Start

### 1. Clone the Repository
```bash
git clone https://github.com/etechwinos2025-pixel/eks_setup.git
cd eks_setup
```

### 2. Initialize Terraform
```bash
terraform init
```

### 3. Review the Plan
```bash
terraform plan
```

### 4. Apply Configuration
```bash
terraform apply
```

The process takes approximately **10-15 minutes**. Once complete, you'll see outputs with:
- Cluster name
- Cluster endpoint
- kubectl configuration command

### 5. Configure kubectl
```bash
aws eks update-kubeconfig --region us-east-1 --name <cluster-name>
kubectl get nodes
```

## Architecture

### Network
- **VPC CIDR**: 10.0.0.0/16
- **Availability Zones**: 3 (auto-selected)
- **Public Subnets**: 10.0.4.0/24, 10.0.5.0/24, 10.0.6.0/24
- **Private Subnets**: 10.0.1.0/24, 10.0.2.0/24, 10.0.3.0/24
- **NAT Gateway**: Single (for cost optimization)

### EKS Cluster
- **Kubernetes Version**: 1.32
- **Endpoint Access**: Public (open to internet, restricted by security groups)
- **Admin Access**: Cluster creator automatically has admin permissions

### Node Groups
| Group | Instance Type | Min | Max | Desired | Purpose |
|-------|---------------|-----|-----|---------|---------|
| primary | t3.small | 1 | 3 | 2 | General workloads |
| secondary | t3.small | 1 | 2 | 1 | Secondary/backup |

## Configuration

### Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `region` | us-east-1 | AWS region |
| `cluster_version` | 1.32 | Kubernetes version |
| `environment` | dev | Environment name (for tagging) |

### Override Defaults
Create `terraform.tfvars`:
```hcl
region           = "us-east-1"
cluster_version  = "1.32"
environment      = "production"
```

## Outputs

After `terraform apply`, you'll get:
- **cluster_name**: EKS cluster name
- **cluster_endpoint**: Kubernetes API endpoint
- **configure_kubectl**: Command to configure kubectl
- **vpc_id**: VPC ID
- **node_security_group_id**: Security group for nodes

## Cost Estimation

| Resource | Cost/Month |
|----------|-----------|
| EKS Cluster | ~$73 |
| 2 t3.small nodes (on-demand) | ~$13 |
| NAT Gateway | ~$32 |
| **Total** | **~$118/month** |

*Costs vary by region and usage. Check AWS Pricing Calculator for exact estimates.*

## Cleanup

To destroy all resources:
```bash
terraform destroy
```

**⚠️ Warning**: This will delete the cluster and all resources. Ensure you've backed up any data.

## Deployment

### Deploy Applications
```bash
# After configuring kubectl
kubectl create deployment nginx --image=nginx:latest
kubectl expose deployment nginx --port=80 --type=LoadBalancer
```

### Monitor Cluster
```bash
kubectl get nodes
kubectl get pods -A
kubectl logs <pod-name>
```

## Troubleshooting

### Cannot connect to cluster
```bash
# Verify cluster is up
aws eks describe-cluster --region us-east-1 --name <cluster-name>

# Reconfigure kubectl
aws eks update-kubeconfig --region us-east-1 --name <cluster-name>
```

### Nodes not joining cluster
```bash
# Check node group status
aws eks describe-nodegroup --cluster-name <cluster-name> --nodegroup-name primary

# Check CloudWatch logs
aws logs tail /aws/eks/<cluster-name>/cluster --follow
```

## Security Considerations

- ✅ Private subnets for nodes (NAT gateway for egress)
- ✅ Security groups restrict traffic between components
- ✅ IAM roles follow least-privilege principle
- ✅ Cluster endpoint is public but restricted by security groups
- ⚠️ Consider using private endpoint + bastion for production

## Useful Commands

```bash
# Get cluster info
kubectl cluster-info
kubectl get componentstatuses

# Describe nodes
kubectl describe nodes

# Check resource usage
kubectl top nodes
kubectl top pods -A

# View events
kubectl get events -A --sort-by='.lastTimestamp'
```

## Support & Documentation

- [AWS EKS Documentation](https://docs.aws.amazon.com/eks/)
- [Terraform AWS Provider](https://registry.terraform.io/providers/hashicorp/aws/latest/docs)
- [Terraform EKS Module](https://registry.terraform.io/modules/terraform-aws-modules/eks/aws/)

## License

MIT

---

Created for simple, production-ready EKS cluster deployments on AWS.