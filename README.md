# AWS EKS Terraform Platform

Terraform configuration for provisioning an AWS-hosted Kubernetes platform with Amazon EKS, VPC networking, managed worker nodes, shared EFS storage, NGINX Ingress, cert-manager, and Argo CD.

> **Environment note:** The checked-in S3 backend and several defaults are account-, region-, and domain-specific. Review the variables and backend configuration before applying them in a different AWS account or region.

## Architecture

- A VPC with two public and two private subnets across two Availability Zones
- An internet gateway for public subnet traffic and NAT gateways for private worker-node egress
- An Amazon EKS control plane with public and private API endpoint access
- An EKS managed node group in the private subnets
- EFS with mount targets, the EFS CSI driver, and an `efs-sc` StorageClass
- NGINX Ingress Controller exposed by an AWS Network Load Balancer
- cert-manager and a Let's Encrypt ClusterIssuer
- Argo CD exposed at a configurable public hostname
- A kubeconfig uploaded to Amazon S3

## Repository layout

| Path | Purpose |
|---|---|
| `00-provider.tf` | Terraform providers, EKS authentication, and S3 backend |
| `01-vpc.tf`–`06-internet-gateway-attach.tf` | VPC, subnets, internet gateway, NAT, and route tables |
| `07-security.tf` | VPC and application security groups |
| `08-keypair.tf` | EC2 key pair and private-key output |
| `09-eks-cluster.tf` | EKS cluster, IAM roles, and cluster administrator access |
| `10-eks-node-group.tf` | Managed node group and worker IAM configuration |
| `11-outputs.tf` | Cluster, ingress, Argo CD, EFS, and kubeconfig outputs |
| `12-cert-manager.tf` | cert-manager Helm release and ClusterIssuer |
| `13-nginx-ingress.tf` | NGINX Ingress Controller Helm release and load balancer lookup |
| `14-efs.tf` | EFS, mount targets, CSI driver, IAM, and StorageClass |
| `15-argocd.tf` | Argo CD Helm release and initial administrator secret |
| `16-kubeconfig.tf` | kubeconfig generation and S3 upload |
| `17-ebs.tf` | AWS EBS CSI add-on configuration |
| `charts/cluster-issuer/` | Local cert-manager ClusterIssuer chart |
| `manifests/argocd-ingress.yaml` | Example Argo CD Ingress resource |
| `create-secret.sh` | Helper for Kubernetes or registry secret creation |
| `efs-test/` | EFS validation resources |
| `jira/` | Jira-related deployment resources |

## Prerequisites

Install and configure:

- Terraform 1.x
- AWS CLI v2
- kubectl
- Helm 3
- AWS credentials with permission to create the referenced VPC, EC2, IAM, EKS, EFS, S3, and Kubernetes resources

Verify AWS identity and region:

```bash
aws sts get-caller-identity
aws configure get region
```

Configure long-lived or temporary AWS credentials using environment variables or a named profile. Do not commit credentials, private keys, or kubeconfigs.

## Before deployment

1. Create the S3 bucket named in `00-provider.tf`, or change the backend block to an existing bucket.
2. Create or enable S3 native state locking as required by the Terraform version and backend setup.
3. Choose an AWS account and region. The default region is `us-west-2` and the default S3 bucket is account-specific.
4. Review `variables.tf` before deployment. In particular, replace or limit:
   - `cluster_admin_arns`
   - `cluster_endpoint_public_access_cidrs`
   - `security_group_ingress_cidr`
   - `argocd_hostname`
   - `letsencrypt_email`
   - `kubec​onfig_bucket` and `kubec​onfig_s3_key`
   - Node group size and instance type values
5. Point the Argo CD and application hostnames at the NGINX Network Load Balancer address after provisioning.

Example variable overrides:

```hcl
aws_region                         = "us-west-2"
cluster_name                       = "pod7cloud-eks"
cluster_endpoint_public_access_cidrs = ["203.0.113.10/32"]
cluster_admin_arns                 = ["arn:aws:iam::123456789012:user/example"]
argocd_hostname                    = "argocd.example.com"
letsencrypt_email                  = "devops@example.com"
node_instance_types                = ["t3.medium"]
node_desired_size                  = 3
node_min_size                      = 3
node_max_size                  = 6
```

Use a `terraform.tfvars` file for environment-specific values. The repository's `.gitignore` should protect state and secret files, but verify it before adding credentials or generated key material.

## Deploy

Initialize and review the plan:

```bash
terraform init
terraform fmt -check
terraform validate
terraform plan
```

Apply the platform:

```bash
terraform apply
```

Terraform will create the network, EKS control plane and managed nodes, storage components, ingress, certificate management, and Argo CD.

## Connect to the cluster

Use the Terraform output:

```bash
aws eks update-kubeconfig \
  --region us-west-2 \
  --name pod7cloud-eks
```

Or copy the generated kubeconfig from S3:

```bash
aws s3 cp s3://<bucket>/eks/kubeconfig ~/.kube/config
kubectl get nodes
kubectl get pods -A
```

The exact bucket and key are available from:

```bash
terraform output kubeconfig_s3_uri
terraform output kubeconfig_download_command
```

## Expose Argo CD

The Argo CD chart disables the chart-managed Ingress; `manifests/argocd-ingress.yaml` provides a resource to apply after DNS is configured.

```bash
kubectl apply -f manifests/argocd-ingress.yaml
terraform output load_balancer_hostname
terraform output argocd_url
```

Create a DNS CNAME record from `argocd_hostname` to the load-balancer hostname. Ensure the hostname matches the value in `variables.tf` and the manifest.

Retrieve the initial Argo CD password:

```bash
terraform output -raw argocd_initial_admin_password
```

Log in as `admin` over HTTPS, then change or remove the initial credential according to your operating policy.

## Verify storage and ingress

```bash
kubectl get storageclass
kubectl get pods -n cert-manager
kubectl get pods -n nginx-ingress
kubectl get pods -n argocd
kubectl get ingress -A
kubectl get svc -n nginx-ingress
```

Use `efs-test/` to validate dynamic EFS provisioning, and confirm that NFS port 2049 is reachable only from the VPC.

## Destroy

Review all resources and remove applications or finalizers first where appropriate:

```bash
terraform plan -destroy
terraform destroy
```

State is stored in S3, and S3-native state locking is configured. Verify that the backend bucket and region exist before any later `terraform init` or destroy operation.

## Security considerations

- Restrict public EKS API endpoint access instead of leaving `0.0.0.0/0` enabled.
- Restrict the application security-group source CIDR.
- Use dedicated IAM identities and least-privilege policies for Terraform and workload access.
- Rotate generated credentials and the Argo CD initial password.
- Protect the S3 bucket, Terraform state, private key, kubeconfig, and Argo CD secret data.
- Enable account-level logging and monitoring for EKS, EFS, load balancers, and AWS API activity.
- Treat the EFS lifecycle transition and encrypted shared filesystem as availability and recovery requirements, not just cost controls.

## Useful Terraform outputs

```bash
terraform output eks_cluster_name
terraform output eks_cluster_endpoint
terraform output kubec​onfig_command
terraform output kube​config_s3_uri
terraform output load_balancer_hostname
terraform output argocd_url
terraform output efs_file_system_id
terraform output efs_storage_class
```
