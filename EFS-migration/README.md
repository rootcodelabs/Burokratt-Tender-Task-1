# EFS Migration Guide

## Prerequisites on Control Machine

### Install Required Ansible Collections

```bash
ansible-galaxy collection install kubernetes.core
ansible-galaxy collection install community.general
ansible-galaxy collection install amazon.aws
```

### Install Python Dependencies

```bash
pip install kubernetes boto3 botocore
```

## Usage

### Full Migration

```bash
ansible-playbook -i inventory/hosts.yml migrate-to-efs.yml
```

### Run Specific Steps

```bash
# Run infrastructure setup (steps 1-3)
ansible-playbook -i inventory/hosts.yml migrate-to-efs.yml --tags step1,step2,step3

# Run migration and deployment update (steps 4-5)
ansible-playbook -i inventory/hosts.yml migrate-to-efs.yml --tags step4,step5

# Run all steps
ansible-playbook -i inventory/hosts.yml migrate-to-efs.yml --tags step1,step2,step3,step4,step5
```

### Dry Run

```bash
ansible-playbook -i inventory/hosts.yml migrate-to-efs.yml --check
```
