# AWS EKS Terraform Platform

This project is our Terraform-based AWS platform for learning and delivering a Kubernetes environment together. It provisions Amazon EKS, VPC networking, managed worker nodes, shared EFS storage, NGINX Ingress, cert-manager, and Argo CD as one reproducible infrastructure stack.

> **Environment note:** Our checked-in S3 backend and several defaults are account-, region-, and domain-specific. Before we deploy from a new AWS account or environment, we should review the variables and backend configuration and replace values that belong to this project’s existing setup.

## Our architecture

We organize the platform around a VPC and a managed Kubernetes control plane:

- We create a VPC with two public and two private subnets across two Availability Zones.
- We use an internet gateway for public subnet traffic and NAT gateways for private worker-node egress.
- We deploy an Amazon EKS control plane with both public and private API endpoint access.
- We place our managed worker-node group in the private subnets.
- We provision encrypted EFS storage with mount targets, the EFS CSI driver, and an `efs-sc` StorageClass.
- We expose applications through the NGINX Ingress Controller using an AWS Network Load Balancer.
- We install cert-manager and a Let's Encrypt ClusterIssuer so we can manage TLS certificates.
- We deploy Argo CD and expose it through a configurable public hostname.
- We generate and upload an administrator kubeconfig to Amazon S3 for later access.

## Repository guide

These files are organized by infrastructure layer. As we work on the project, this is where we can find and change each concern:

| Path | Purpose |
|---|---|
| `00-provider.tf` | Terraform providers, EKS authentication, and our S3 backend |
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

## What we need

Before we begin, we install and configure:

- Terraform 1.x
- AWS CLI v2
- kubectl
- Helm 3
- AWS credentials with permission to create the referenced VPC, EC2, IAM, EKS, EFS, S3, and Kubernetes resources

We can verify our AWS identity and region with:

```bash
aws sts get-caller-identity
aws configure get region
```

We should use environment variables, a named AWS profile, or temporary credentials. We must not commit credentials, private keys, or kubeconfigs to the repository.

## Preparing our deployment

Before we apply this stack, we should:

1. Create the S3 bucket named in `00-provider.tf`, or change the backend block to an existing bucket we control.
2. Confirm that S3 state locking is configured according to our Terraform version and backend setup.
3. Choose our target AWS account and region. The default is `us-west-2` and the default S3 bucket is account-specific.
4. Review `variables.tf` and replace or limit these values when needed:
   - `cluster_admin_arns`
   - `cluster_endpoint_public_access_cidrs`
   - `security_group_ingress_cidr`
   - `argocd_hostname`
   - `letsencrypt_email`
   - `kubec​onfig_bucket` and `kubec​onfig_s3_key`
   - Node group size and instance type values
5. Plan to point our Argo CD and application hostnames at the NGINX Network Load Balancer after provisioning.

An example set of environment-specific overrides might look like:

```hcl
aws_region                          = "us-west-2"
cluster_name                        = "pod7cloud-eks"
cluster_endpoint_public_access_cidrs = ["203.0.113.10/32"]
cluster_admin_arns                  = ["arn:aws:iam::123456789012:user/example"]
argocd_hostname                     = "argocd.example.com"
letsencrypt_email                   = "devops@example.com"
node_instance_types                 = ["t3.medium"]
node_desired_size                   = 3
node_min_size                       = 3
node_max_size                       = 6
```

We recommend keeping environment-specific values in a `terraform.tfvars` file. The repository's `.gitignore` is intended to protect state and secret files, but we should verify its coverage before adding credentials or generated key material.

## Deploying our platform

We begin by initializing Terraform and reviewing exactly what it proposes:

```bash
terraform init
terraform fmt -check
terraform validate
terraform plan
```

When we are comfortable with the plan, we apply the platform:

```bash
terraform apply
```

Terraform will create our network, EKS control plane and managed nodes, storage components, ingress, certificate management, and Argo CD resources.

## Connecting to our cluster

We can use the Terraform-provided command to configure kubectl:

```bash
aws eks update-kubeconfig \
  --region us-west-2 \
  --name pod7cloud-eks
```

Alternatively, we can copy the generated kubeconfig from S3:

```bash
aws s3 cp s3://<bucket>/eks/kubeconfig ~/.kube/config
kubectl get nodes
kubectl get pods -A
```

We can find the exact bucket and key through our Terraform outputs:

```bash
terraform output kubeconfig_s3_uri
terraform output kubeconfig_download_command
```

## Exposing Argo CD

The Argo CD chart disables its chart-managed Ingress. We provide `manifests/argocd-ingress.yaml` as an example resource to apply after our DNS is configured.

```bash
kubectl apply -f manifests/argocd-ingress.yaml
terraform output load_balancer_hostname
terraform output argocd_url
```

We then create a DNS CNAME record from `argocd_hostname` to the load-balancer hostname. Our hostname should match the value in `variables.tf` and the manifest.

We can retrieve the initial Argo CD password with:

```bash
terraform output -raw argocd_initial_admin_password
```

We log in as `admin` over HTTPS, then change or remove the initial credential according to our operating policy.

## Checking our services

These commands help us verify storage, ingress, certificates, and the core platform services:

```bash
kubectl get storageclass
kubectl get pods -n cert-manager
kubectl get pods -n nginx-ingress
kubectl get pods -n argocd
kubectl get ingress -A
kubectl get svc -n nginx-ingress
```

We can use `efs-test/` to validate dynamic EFS provisioning. We should also confirm that NFS port 2049 is reachable only from within our VPC.

## Cleaning up our environment

Before we destroy the platform, we should review the plan and remove applications or finalizers where appropriate:

```bash
terraform plan -destroy
terraform destroy
```

Our state is stored in S3 with S3-native state locking. We should verify that the backend bucket and region exist before any future `terraform init` or destroy operation.

## Our security responsibilities

- We restrict public EKS API endpoint access instead of leaving `0.0.0.0/0` enabled.
- We restrict the application security-group source CIDR.
- We use dedicated IAM identities and least-privilege policies for Terraform and workload access.
- We rotate generated credentials and the Argo CD initial password.
- We protect the S3 bucket, Terraform state, private key, kubeconfig, and Argo CD secret data.
- We enable account-level logging and monitoring for EKS, EFS, load balancers, and AWS API activity.
- We treat EFS lifecycle transitions and our encrypted shared filesystem as availability and recovery requirements, not only cost controls.

## Our useful Terraform outputs

```bash
terraform output eks_cluster_name
terraform output eks_cluster_endpoint
terraform output kube​config_command
terraform output kube​config_s3_uri
terraform output load_balancer_hostname
terraform output argocd_url
terraform output efs_file_system_id
terraform output efs_storage_class
```
