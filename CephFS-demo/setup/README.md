# CephFS Setup - Rook-Ceph Cluster Configuration

## Overview
This directory contains the foundational Rook-Ceph cluster and CephFS filesystem configuration required before deploying applications.

## Components

### 1. ceph-cluster.yaml
Defines the Rook-Ceph cluster with:
- **Ceph Version**: v18.2.4
- **Monitors**: 3 replicas
- **Managers**: 2 replicas
- **Dashboard**: Enabled (non-SSL)
- **Storage**: Device-based (`nvme1n1`)

### 2. cephfs.yaml
Defines the CephFS filesystem with:
- **Metadata Pool**: 3 replicas
- **Data Pool**: 3 replicas
- **MDS Servers**: 1 active with standby

## Prerequisites
- Kubernetes cluster (1.19+)
- Rook operator installed
- Available block devices on nodes (`nvme1n1` or modify accordingly)
- Minimum 3 nodes for Ceph cluster

## Installation Steps

### Step 1: Install Rook Operator
```bash
kubectl apply -f https://raw.githubusercontent.com/rook/rook/release-1.13/deploy/examples/crds.yaml
kubectl apply -f https://raw.githubusercontent.com/rook/rook/release-1.13/deploy/examples/common.yaml
kubectl apply -f https://raw.githubusercontent.com/rook/rook/release-1.13/deploy/examples/operator.yaml
```

### Step 2: Verify Operator is Running
```bash
kubectl get pods -n rook-ceph -l app=rook-ceph-operator
```

### Step 3: Deploy Ceph Cluster
```bash
kubectl apply -f ceph-cluster.yaml
```

### Step 4: Wait for Cluster to be Ready
```bash
kubectl get cephcluster -n rook-ceph -w
```

Expected output: `HEALTH: HEALTH_OK`

### Step 5: Deploy CephFS Filesystem
```bash
kubectl apply -f cephfs.yaml
```

### Step 6: Verify CephFS is Active
```bash
kubectl get cephfilesystem -n rook-ceph
```

## Validation

### Check Cluster Health
```bash
kubectl exec -n rook-ceph deploy/rook-ceph-tools -- ceph status
```

### Check OSD Status
```bash
kubectl exec -n rook-ceph deploy/rook-ceph-tools -- ceph osd status
```

### Check MDS Status
```bash
kubectl exec -n rook-ceph deploy/rook-ceph-tools -- ceph mds stat
```

### Access Ceph Dashboard
```bash
# Get dashboard service
kubectl get svc -n rook-ceph rook-ceph-mgr-dashboard

# Port forward to access locally
kubectl port-forward -n rook-ceph svc/rook-ceph-mgr-dashboard 8443:8443

# Get admin password
kubectl -n rook-ceph get secret rook-ceph-dashboard-password -o jsonpath="{['data']['password']}" | base64 --decode && echo
```

## Configuration Customization

### Modify Storage Devices
Edit [ceph-cluster.yaml](ceph-cluster.yaml) line 27:
```yaml
devices:
  - name: "nvme1n1"  # Change to your device name
```

To find available devices:
```bash
lsblk -f
```

### Adjust Resource Limits
Modify resource requests/limits in [ceph-cluster.yaml](ceph-cluster.yaml):
```yaml
resources:
  osd:
    limits:
      memory: "2Gi"  # Adjust based on your cluster capacity
```

### Change Replication Factor
Edit [cephfs.yaml](cephfs.yaml):
```yaml
metadataPool:
  replicated:
    size: 3  # Minimum 2, recommended 3
```

## Troubleshooting

### OSD Pods Not Starting
- Verify devices are available: `lsblk`
- Check device name matches configuration
- Ensure devices are not already in use

### Cluster Stuck in HEALTH_WARN
```bash
kubectl exec -n rook-ceph deploy/rook-ceph-tools -- ceph health detail
```

### MDS Not Active
```bash
kubectl logs -n rook-ceph -l app=rook-ceph-mds
```

## Next Steps
After successful setup, deploy applications using the Helm chart:
```bash
cd ../cephfs-helm-chart
helm install cephfs-app .
```

## Resource Requirements

| Component | CPU Request | Memory Request | CPU Limit | Memory Limit |
|-----------|-------------|----------------|-----------|--------------|
| Monitor   | 100m        | 256Mi          | -         | 1Gi          |
| Manager   | 100m        | 256Mi          | -         | 1Gi          |
| OSD       | 500m        | 1Gi            | -         | 2Gi          |
| MDS       | 500m        | 1Gi            | -         | 2Gi          |

## Uninstall

**Warning**: This will delete all data!

```bash
kubectl delete -f cephfs.yaml
kubectl delete -f ceph-cluster.yaml
kubectl delete namespace rook-ceph
```

## References
- [Rook Documentation](https://rook.io/docs/rook/latest/)
- [Ceph Documentation](https://docs.ceph.com/)
