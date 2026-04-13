# Setup NFS Provisioner Role

This Ansible role deploys an in-cluster NFS provisioner for OpenShift, providing dynamic storage provisioning using hostPath storage on a worker node.

## Description

This role automates the deployment of an NFS provisioner that:
- Runs inside the OpenShift cluster
- Uses hostPath storage (`/srv/nfs`) on a worker node
- Provides dynamic PersistentVolume provisioning
- Creates a StorageClass for use by applications
- Can be set as the default storage class

## Requirements

- OpenShift cluster (SNO or multi-node)
- `oc` CLI installed and configured
- Cluster-admin privileges
- Python kubernetes module installed

## Role Variables

Available variables with default values (see `defaults/main.yaml`):

```yaml
# NFS provisioner name (used in StorageClass)
nfs_provisioner_name: example.com/nfs

# Storage class name to create
nfs_storage_class_name: nfs

# Set as default storage class
set_as_default: true
```

## Dependencies

- `kubernetes.core` Ansible collection

## Example Playbook

### Standalone Usage

```yaml
---
- name: Setup NFS Provisioner
  hosts: bastion
  become: false
  gather_facts: true
  
  tasks:
    - name: Setup NFS provisioner
      include_role:
        name: setup_nfs_provisioner
```

### With Custom Variables

```yaml
---
- name: Setup NFS Provisioner with custom settings
  hosts: bastion
  become: false
  gather_facts: true
  
  tasks:
    - name: Setup NFS provisioner
      include_role:
        name: setup_nfs_provisioner
      vars:
        nfs_storage_class_name: my-nfs
        set_as_default: false
```

### Integrated with Logging Stack

```yaml
---
- name: Deploy Logging Stack with NFS
  hosts: bastion
  become: false
  gather_facts: true
  
  tasks:
    - name: Setup NFS provisioner
      include_role:
        name: setup_nfs_provisioner
    
    - name: Deploy logging stack
      include_role:
        name: deploy_logging_stack
      vars:
        nfs_storage_class: nfs
```

## What Gets Deployed

### Namespace
- `nfs-provisioner` - Dedicated namespace for NFS provisioner

### Resources Created
1. **ServiceAccount**: `nfs-provisioner`
2. **Service**: NFS service with multiple ports (NFS, mountd, rpcbind, etc.)
3. **Deployment**: NFS provisioner pod
4. **SecurityContextConstraints**: Custom SCC for NFS provisioner
5. **ClusterRole**: Permissions for managing PVs and PVCs
6. **ClusterRoleBinding**: Binds ClusterRole to ServiceAccount
7. **Role**: Leader election permissions
8. **RoleBinding**: Binds Role to ServiceAccount
9. **StorageClass**: Dynamic provisioning storage class

### Storage Location
- **Host Path**: `/srv/nfs` on the worker node
- **Init Container**: Sets up directory with proper SELinux context and permissions

## Architecture

```
Application PVC Request
        ↓
   StorageClass (nfs)
        ↓
NFS Provisioner Pod
        ↓
  Creates PV on /srv/nfs
        ↓
   Binds PVC to PV
```

## Verification

After running the role, verify the deployment:

```bash
# Check namespace
oc get namespace nfs-provisioner

# Check pods
oc get pods -n nfs-provisioner

# Expected output:
# NAME                               READY   STATUS    RESTARTS   AGE
# nfs-provisioner-xxxxxxxxxx-xxxxx   1/1     Running   0          2m

# Check storage class
oc get storageclass

# Expected output:
# NAME   PROVISIONER          RECLAIMPOLICY   VOLUMEBINDINGMODE
# nfs    example.com/nfs      Delete          Immediate

# Test with a PVC
cat <<EOF | oc apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-nfs
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
  storageClassName: nfs
EOF

# Check if PVC is bound
oc get pvc test-nfs

# Clean up test
oc delete pvc test-nfs
```

## Node Selection

The role automatically:
1. Finds the first worker node in the cluster
2. If no worker node exists (SNO), uses the first available node
3. Annotates the namespace to schedule the provisioner on that node

For SNO clusters, the provisioner runs on the control-plane node.

## Storage Capacity

The available storage is limited by:
- The disk space available on `/srv/nfs` on the host
- The worker node's total disk capacity

Monitor disk usage:
```bash
# SSH to the node and check
df -h /srv/nfs
```

## Security

### SELinux Context
The init container sets the proper SELinux context:
```bash
chcon -Rt svirt_sandbox_file_t /srv/nfs
```

### Capabilities
The provisioner requires:
- `DAC_READ_SEARCH` - Read files
- `SYS_RESOURCE` - Manage resources

### SecurityContextConstraints
A custom SCC is created with minimal required privileges.

## Troubleshooting

### Issue: Provisioner pod not starting

```bash
# Check pod status
oc get pods -n nfs-provisioner

# Check pod logs
oc logs -n nfs-provisioner deployment/nfs-provisioner

# Check events
oc get events -n nfs-provisioner --sort-by='.lastTimestamp'
```

### Issue: PVC stuck in Pending

```bash
# Check PVC status
oc describe pvc <pvc-name>

# Check provisioner logs
oc logs -n nfs-provisioner deployment/nfs-provisioner

# Verify storage class
oc get storageclass nfs -o yaml
```

### Issue: Permission denied errors

```bash
# Check SELinux context on the node
ssh <node> ls -lZ /srv/nfs

# Should show: svirt_sandbox_file_t

# If not, manually fix:
ssh <node> sudo chcon -Rt svirt_sandbox_file_t /srv/nfs
```

### Issue: Out of disk space

```bash
# Check disk usage on node
ssh <node> df -h /srv/nfs

# Clean up old PVs if needed
oc get pv | grep Released
oc delete pv <pv-name>
```

## Cleanup

To remove the NFS provisioner:

```bash
# Delete all PVCs using the storage class first
oc get pvc --all-namespaces | grep nfs

# Delete the storage class
oc delete storageclass nfs

# Delete the namespace (this removes all resources)
oc delete namespace nfs-provisioner

# Clean up SCC
oc delete scc nfs-provisioner

# Clean up RBAC
oc delete clusterrole nfs-provisioner-runner
oc delete clusterrolebinding run-nfs-provisioner
```

## Limitations

- **Single node storage**: Data is stored on one node only
- **No replication**: If the node fails, data is unavailable
- **Capacity**: Limited by node disk space
- **Performance**: Depends on node disk I/O

## Best Practices

1. **Monitor disk usage**: Set up alerts for `/srv/nfs` disk usage
2. **Backup important data**: NFS provisioner doesn't provide backup
3. **Use for development/testing**: For production, consider external NFS or other storage solutions
4. **Set resource limits**: Configure PVC size limits in your applications

## License

Apache License 2.0

## Author Information

This role was created as part of the OpenShift logging stack deployment automation.