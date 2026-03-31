# Amazon Web Services (AWS)

## Prerequisites

- AWS account with appropriate IAM permissions
- `aws` CLI configured with credentials
- VPC with subnets (public or private with NAT)
- SSH key pair (for EC2 creation, not for Talos access — Talos has no SSH)
- Knowledge of target region and availability zones

## Image Provisioning

Talos publishes official AMIs. Find the latest for your region:

```bash
# List official Talos AMIs
aws ec2 describe-images \
  --owners 540036508848 \
  --filters "Name=name,Values=talos-v1.9.*" \
  --query 'Images | sort_by(@, &CreationDate) | [-1]' \
  --region us-east-1
```

For custom extensions, use Image Factory to get an AMI ID or build from the raw image:

```bash
# Download raw AWS image
wget https://factory.talos.dev/image/<schematic-id>/v1.9.0/aws-amd64.raw.xz

# Import as AMI (requires S3 bucket)
xz -d aws-amd64.raw.xz
aws s3 cp aws-amd64.raw s3://my-bucket/talos/
aws ec2 import-image \
  --disk-containers "Format=raw,UserBucket={S3Bucket=my-bucket,S3Key=talos/aws-amd64.raw}"
```

## Network Requirements

**Security Groups:**

Control Plane SG:
| Port | Protocol | Source | Purpose |
|---|---|---|---|
| 50000 | TCP | Talos nodes + admin | Talos API |
| 50001 | TCP | Talos nodes | trustd |
| 6443 | TCP | All nodes + admin | K8s API |
| 2379-2380 | TCP | Control plane nodes | etcd |
| 10250 | TCP | All nodes | kubelet |

Worker SG:
| Port | Protocol | Source | Purpose |
|---|---|---|---|
| 50000 | TCP | Talos nodes + admin | Talos API |
| 50001 | TCP | Talos nodes | trustd |
| 10250 | TCP | All nodes | kubelet |

## Machine Config Specifics

```yaml
machine:
  install:
    disk: /dev/xvda    # AWS EBS root volume
    image: ghcr.io/siderolabs/installer:v1.9.0
  kubelet:
    extraArgs:
      cloud-provider: external
  certSANs:
    - <nlb-dns-name>
cluster:
  controlPlane:
    endpoint: https://<nlb-dns-name>:6443
  externalCloudProvider:
    enabled: true
    manifests:
      - https://raw.githubusercontent.com/kubernetes/cloud-provider-aws/master/releases/latest/cloud-controller-manager-daemonset.yaml
```

**Important:** AWS instance types using NVMe storage (most modern types) use `/dev/nvme0n1` instead of `/dev/xvda`.

## Control Plane Endpoint

**Network Load Balancer (NLB) — Recommended:**

```bash
# Create NLB
aws elbv2 create-load-balancer \
  --name talos-cp \
  --type network \
  --subnets subnet-xxx subnet-yyy subnet-zzz \
  --scheme internal

# Create target group
aws elbv2 create-target-group \
  --name talos-cp-6443 \
  --protocol TCP \
  --port 6443 \
  --vpc-id vpc-xxx \
  --target-type instance \
  --health-check-protocol TCP \
  --health-check-port 6443

# Register CP instances
aws elbv2 register-targets \
  --target-group-arn <tg-arn> \
  --targets Id=i-cp1 Id=i-cp2 Id=i-cp3

# Create listener
aws elbv2 create-listener \
  --load-balancer-arn <nlb-arn> \
  --protocol TCP \
  --port 6443 \
  --default-actions Type=forward,TargetGroupArn=<tg-arn>
```

Use the NLB DNS name as the cluster endpoint.

## Provisioning Commands

```bash
# Launch control plane instances
for i in 1 2 3; do
  aws ec2 run-instances \
    --image-id ami-xxxxxxxxx \
    --instance-type t3.xlarge \
    --count 1 \
    --subnet-id subnet-xxx \
    --security-group-ids sg-cp-xxx \
    --block-device-mappings "DeviceName=/dev/xvda,Ebs={VolumeSize=50,VolumeType=gp3}" \
    --tag-specifications "ResourceType=instance,Tags=[{Key=Name,Value=talos-cp${i}}]"
done

# Launch worker instances
for i in 1 2 3; do
  aws ec2 run-instances \
    --image-id ami-xxxxxxxxx \
    --instance-type t3.2xlarge \
    --count 1 \
    --subnet-id subnet-xxx \
    --security-group-ids sg-worker-xxx \
    --block-device-mappings "DeviceName=/dev/xvda,Ebs={VolumeSize=100,VolumeType=gp3}" \
    --tag-specifications "ResourceType=instance,Tags=[{Key=Name,Value=talos-worker${i}}]"
done
```

**Pass machine config via user data:**

```bash
aws ec2 run-instances \
  --image-id ami-xxxxxxxxx \
  --instance-type t3.xlarge \
  --user-data file://controlplane.yaml \
  ...
```

## Platform-Specific Gotchas

- **Disk device names**: Modern instance types use NVMe (`/dev/nvme0n1`). Older types use `/dev/xvda`. Check your instance type.
- **IAM for CCM**: The AWS Cloud Controller Manager needs an IAM role with permissions for EC2, ELB, and Route53. Attach via instance profile.
- **No VIP**: AWS does not support gratuitous ARP. Use NLB instead of Talos VIP for the control plane endpoint.
- **Instance metadata**: Talos v1.9 supports IMDSv2. No special configuration needed.
- **EBS volumes**: For storage CSI, attach additional EBS volumes to worker nodes. The AWS EBS CSI driver handles dynamic provisioning.
- **Placement groups**: Use spread placement groups for control plane nodes across AZs.
- **Graviton (ARM)**: Talos supports ARM64. Use `arm64` image variants with Graviton instance types for cost savings.
