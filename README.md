# Kubernetes Storage Migration: Longhorn to AWS EFS

## Executive Summary

This solution provides a fully automated, production-grade migration path for transitioning Kubernetes persistent storage from Longhorn to AWS Elastic File System (EFS). Designed for enterprise environments, it orchestrates the complete lifecycle of infrastructure provisioning, data migration, and application deployment updates with zero data loss and minimal downtime.

**Key Benefits:**
- 🔄 **Automated End-to-End Migration**: Five-step orchestrated process with validation at each stage
- 🛡️ **Zero Data Loss**: Rsync-based migration with integrity verification
- ☁️ **Cloud-Native**: Leverages AWS EFS for scalable, managed shared storage
- 🎯 **Modular Architecture**: Tag-based execution allows step-by-step or full automation
- 📊 **Comprehensive Validation**: Pre-flight checks, post-migration verification, and detailed logging

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                    Migration Orchestrator                        │
│                     (Ansible Playbook)                          │
└────────────┬────────────────────────────────────────────────────┘
             │
             ├─► Step 1: EFS CSI Driver Installation (Helm)
             │
             ├─► Step 2: Infrastructure Provisioning (Terraform)
             │            ├─ EFS Filesystem
             │            ├─ Security Groups
             │            └─ Mount Targets (Multi-AZ)
             │
             ├─► Step 3: StorageClass Configuration
             │            └─ Dynamic Provisioning Setup
             │
             ├─► Step 4: Data Migration
             │            ├─ Scale Down Applications
             │            ├─ Create EFS PVCs
             │            ├─ Rsync Jobs (Longhorn → EFS)
             │            └─ Integrity Verification
             │
             └─► Step 5: Deployment Updates
                          ├─ Update PVC References
                          ├─ Scale Up Applications
                          └─ Health Verification
```

---

## Features

### Infrastructure Management
- ✅ Automated EFS filesystem provisioning with Terraform
- ✅ Multi-AZ mount target deployment for high availability
- ✅ Security group configuration with least-privilege access
- ✅ Support for encrypted EFS filesystems

### Data Migration
- ✅ Application-aware migration (scales down before data copy)
- ✅ Rsync-based transfer with progress tracking
- ✅ File count verification to ensure data integrity
- ✅ Parallel migration support for multiple applications

### Safety & Validation
- ✅ Pre-flight checks for required tools (kubectl, helm, terraform, aws-cli)
- ✅ AWS credential validation
- ✅ Kubernetes context verification
- ✅ PVC binding verification before migration
- ✅ Post-deployment health checks
- ✅ Configurable timeouts and retry logic

### Operational Features
- ✅ Tag-based execution for granular control
- ✅ Dry-run mode for testing
- ✅ Detailed migration logs and summaries
- ✅ Support for multiple namespaces and applications

---

## Prerequisites

### Control Machine Requirements

**Required Tools:**
- Ansible 2.9+
- Python 3.8+
- kubectl 1.20+
- helm 3.0+
- terraform 1.0+
- aws-cli 2.0+

**Install Ansible Collections:**
```bash
ansible-galaxy collection install kubernetes.core
ansible-galaxy collection install community.general
ansible-galaxy collection install amazon.aws
```

**Install Python Dependencies:**
```bash
pip install kubernetes boto3 botocore
```

### AWS Requirements

**IAM Permissions:**
The executing AWS identity requires:
- EC2: Create/manage security groups
- EFS: Create/manage file systems and mount targets
- IAM: Create service accounts (if using IRSA)

**AWS Configuration:**
```bash
aws configure
# Ensure credentials are set for target account/region
aws sts get-caller-identity  # Verify access
```

### Kubernetes Cluster Requirements

- EKS cluster (recommended) or self-managed Kubernetes on AWS EC2
- Existing Longhorn installation with ReadWriteMany (RWX) PVCs
- Network connectivity between worker nodes and EFS mount targets (NFS port 2049)
- kubectl configured with cluster access

---

## Configuration

### 1. Update AWS Configuration

Edit `group_vars/all.yml`:

```yaml
# AWS Settings
aws_region: "us-east-1"           # Your AWS region
vpc_id: "vpc-0123456789abcdef0"   # VPC where EKS cluster runs
subnet_ids:                        # Private subnets (multi-AZ recommended)
  - "subnet-0abc123def456789a"
  - "subnet-0def456abc789012b"
  - "subnet-0ghi789jkl012345c"
worker_security_group_id: "sg-0123456789abcdef0"  # EKS node security group
```

### 2. Configure Target Applications

Define applications to migrate in `group_vars/all.yml`:

```yaml
apps_to_migrate:
  - name: "trainbot"                     # Application identifier
    namespace: "production"              # Kubernetes namespace
    old_pvc: "pvc-trainbot-models"      # Source Longhorn PVC
    new_pvc: "trainbot-models-efs"      # Target EFS PVC (will be created)
    storage_size: "500Mi"               # EFS PVC size
    deployment_name: "trainbot"         # Deployment to update
    replicas: 3                         # Original replica count
    mount_path: "/models"               # Volume mount path
    container_name: "trainbot"          # Container name
    image: "example/trainbot:latest"    # Container image
```

### 3. Adjust EFS Settings (Optional)

```yaml
# EFS Configuration
efs_performance_mode: "generalPurpose"  # or "maxIO" for high throughput
efs_throughput_mode: "bursting"         # or "provisioned"
efs_encrypted: true                     # Enable encryption at rest
efs_name: "kubernetes-shared-storage"   # EFS filesystem name

# Migration Timeouts
migration_timeout: 3600      # Maximum time per migration job (seconds)
validation_wait_time: 120    # Pod readiness wait time (seconds)
```

---

## Usage

### Pre-Migration Checklist

- [ ] Review and update `group_vars/all.yml` with your environment details
- [ ] Verify AWS credentials: `aws sts get-caller-identity`
- [ ] Verify kubectl access: `kubectl get nodes`
- [ ] Backup existing data (recommended)
- [ ] Review Longhorn PVC names: `kubectl get pvc -n <namespace>`
- [ ] Verify worker node security group ID

### Execution Options

#### **Option 1: Full Automated Migration**
```bash
ansible-playbook -i inventory/hosts.yml migrate-to-efs.yml
```

#### **Option 2: Step-by-Step Migration**

**Phase 1: Infrastructure Setup**
```bash
ansible-playbook -i inventory/hosts.yml migrate-to-efs.yml \
  --tags step1,step2,step3
```

**Phase 2: Data Migration and Deployment**
```bash
ansible-playbook -i inventory/hosts.yml migrate-to-efs.yml \
  --tags step4,step5
```

**Individual Steps:**
```bash
# Step 1: Install EFS CSI Driver
ansible-playbook -i inventory/hosts.yml migrate-to-efs.yml --tags step1

# Step 2: Provision EFS Infrastructure
ansible-playbook -i inventory/hosts.yml migrate-to-efs.yml --tags step2

# Step 3: Create StorageClass
ansible-playbook -i inventory/hosts.yml migrate-to-efs.yml --tags step3

# Step 4: Migrate Data
ansible-playbook -i inventory/hosts.yml migrate-to-efs.yml --tags step4

# Step 5: Update Deployments
ansible-playbook -i inventory/hosts.yml migrate-to-efs.yml --tags step5
```

#### **Option 3: Dry Run (Validation Only)**
```bash
ansible-playbook -i inventory/hosts.yml migrate-to-efs.yml --check
```

#### **Option 4: Verbose Mode (Debugging)**
```bash
ansible-playbook -i inventory/hosts.yml migrate-to-efs.yml -vvv
```

---

## Migration Process Details

### Step 1: EFS CSI Driver Installation
- Adds AWS EFS CSI Driver Helm repository
- Deploys controller and node DaemonSets to `kube-system`
- Configures service accounts for AWS API access

**Verification:**
```bash
kubectl get pods -n kube-system -l app.kubernetes.io/name=aws-efs-csi-driver
```

### Step 2: Infrastructure Provisioning
- Generates Terraform configuration from Jinja2 template
- Creates EFS filesystem with specified performance mode
- Configures security group allowing NFS (2049) from worker nodes
- Creates mount targets in all specified subnets
- Outputs EFS filesystem ID for StorageClass

**Verification:**
```bash
aws efs describe-file-systems --region <region>
```

### Step 3: StorageClass Creation
- Creates Kubernetes StorageClass named `aws-efs-rwx`
- Configures dynamic provisioning with subdirectories

**Verification:**
```bash
kubectl get storageclass aws-efs-rwx
```

### Step 4: Data Migration
1. Creates new EFS-backed PVCs for each application
2. Waits for PVCs to bind (confirms EFS mount success)
3. Scales down source deployments to 0 replicas
4. Launches Kubernetes Jobs running rsync to copy data
5. Monitors job completion with configurable timeout
6. Verifies file counts match between source and destination

**Monitoring Migration:**
```bash
# Watch migration job status
kubectl get jobs -n <namespace> -w

# View migration logs
kubectl logs -n <namespace> job/migrate-<app-name>-data -f
```

### Step 5: Deployment Updates
1. Updates deployment volume definitions to use new EFS PVCs
2. Scales deployments back to original replica counts
3. Waits for all pods to reach Running state
4. Verifies container readiness
5. Confirms new PVC is mounted correctly

**Verification:**
```bash
# Check deployment status
kubectl get deployments -n <namespace>

# Verify PVC mounts
kubectl get pods -n <namespace>
kubectl describe pod <pod-name> -n <namespace>
```

---

## Post-Migration

### Validation Steps

1. **Verify Application Functionality:**
   ```bash
   # Check pod logs
   kubectl logs -n <namespace> deployment/<deployment-name>
   
   # Test application endpoints
   curl <application-endpoint>
   ```

2. **Verify EFS Mount:**
   ```bash
   kubectl exec -n <namespace> deployment/<deployment-name> -- df -h /mount/path
   ```

3. **Review Migration Summary:**
   Check Ansible playbook output for:
   - EFS Filesystem ID
   - Migrated applications list
   - PVC mappings (old → new)

### Cleanup (After Successful Migration)

**Optional: Remove Longhorn PVCs**
```bash
# Only after confirming data integrity!
kubectl delete pvc <old-pvc-name> -n <namespace>
```

**Terraform State:**
EFS infrastructure state is stored in `/tmp/efs-terraform/` by default. For production:
- Move to remote backend (S3 + DynamoDB)
- Update `terraform_state_backend` in configuration

---

## Troubleshooting

### Common Issues

#### 1. CSI Driver Installation Fails
**Symptom:** Helm installation times out
**Solution:**
```bash
# Check pod status
kubectl get pods -n kube-system | grep efs

# View logs
kubectl logs -n kube-system <efs-csi-controller-pod>

# Verify IAM permissions if using IRSA
```

#### 2. PVC Stuck in Pending
**Symptom:** EFS PVCs don't bind
**Solution:**
```bash
# Check PVC events
kubectl describe pvc <pvc-name> -n <namespace>

# Verify EFS mount targets are active
aws efs describe-mount-targets --file-system-id <fs-id>

# Check security group rules
aws ec2 describe-security-groups --group-ids <sg-id>
```

#### 3. Migration Job Fails
**Symptom:** Rsync job exits with error
**Solution:**
```bash
# Check job logs
kubectl logs -n <namespace> job/migrate-<app-name>-data

# Common causes:
# - Insufficient permissions
# - PVC not mounted
# - Out of disk space
```

#### 4. Deployment Won't Scale Up
**Symptom:** Pods stuck in pending after migration
**Solution:**
```bash
# Check pod events
kubectl describe pod <pod-name> -n <namespace>

# Verify PVC exists and is bound
kubectl get pvc -n <namespace>
```

---

## Security Considerations

- **Encryption:** EFS filesystems are encrypted at rest by default (configurable)
- **Network Security:** NFS access restricted to worker node security group only
- **IAM Permissions:** Follow least-privilege principle for AWS credentials
- **Kubernetes RBAC:** Ensure service accounts have minimal required permissions
- **Data in Transit:** Consider enabling EFS encryption in transit for sensitive data

---

## Performance Tuning

### For High-Throughput Workloads
```yaml
efs_performance_mode: "maxIO"
efs_throughput_mode: "provisioned"
```

### For Large-Scale Migrations
```yaml
migration_timeout: 7200  # Increase for large datasets
```

### For Multiple Concurrent Applications
Add more applications to `apps_to_migrate` list - migrations run in parallel.

---

## Rollback Procedure

If issues are detected post-migration:

1. **Immediate Rollback:**
   ```bash
   # Revert to Longhorn PVCs
   kubectl set volume deployment/<name> -n <namespace> \
     --add --name=storage-volume --type=pvc \
     --claim-name=<old-longhorn-pvc> --mount-path=<path> --overwrite
   ```

2. **Investigate Issues:**
   - Review migration job logs
   - Check EFS mount target health
   - Verify data integrity

3. **Reattempt Migration:**
   - Address identified issues
   - Rerun specific migration steps using tags


## Technical Specifications

| Component | Version | Purpose |
|-----------|---------|---------|
| Ansible | 2.9+ | Orchestration engine |
| Terraform | 1.0+ | Infrastructure as Code |
| AWS EFS CSI Driver | Latest | Kubernetes storage provisioner |
| Rsync | 3.x | Data synchronization |


