# Deploy Logging Stack Role

This Ansible role deploys a complete logging stack on OpenShift using Loki, including:
- Loki Operator
- Cluster Logging Operator
- Local Storage Operator
- LokiStack with S3 (MinIO) backend
- ClusterLogForwarder for log collection
- Cluster Observability Operator
- Logging UI Plugin

## Requirements

- OpenShift cluster (SNO or multi-node) already deployed
- `oc` CLI installed and configured
- Access to OpenShift cluster with cluster-admin privileges
- MinIO server deployed and accessible
- NFS storage configured and available
- Python kubernetes module installed on the bastion host

## Role Variables

Available variables are listed below, along with default values (see `defaults/main.yaml`):

```yaml
# Logging namespace
logging_namespace: openshift-logging

# LokiStack configuration
lokistack_name: lokistack-sample
loki_size: 1x.extra-small  # Options: 1x.extra-small, 1x.small, 1x.medium

# Storage configuration
nfs_storage_class: nfs

# MinIO configuration
minio_ip: ""
minio_port: 9000
minio_region: us-east-1
minio_bucket: test
minio_access_key: ""
minio_secret_key: ""

# ServiceAccount for log collection
log_collector_sa: logcollector
```

## Dependencies

This role requires:
- `kubernetes.core` Ansible collection
- OpenShift cluster with internet connectivity (for operator installation)
- MinIO server for S3 object storage
- NFS server for persistent storage

## Example Playbook

```yaml
---
- name: Deploy OpenShift Logging Stack
  hosts: bastion
  become: false
  gather_facts: true
  
  tasks:
    - name: Include deploy_logging_stack role
      include_role:
        name: deploy_logging_stack
      vars:
        minio_ip: "192.168.1.100"
        minio_access_key: "minioadmin"
        minio_secret_key: "minioadmin"
        minio_bucket: "loki-logs"
        nfs_storage_class: "nfs"
        loki_size: "1x.extra-small"
```

## Usage

### Using the standalone playbook:

```bash
cd /path/to/Ansible-OpenShift-Provisioning
ansible-playbook playbooks/deploy_logging_stack_master.yaml
```

The playbook will prompt you for:
- MinIO Server IP
- MinIO Access Key
- MinIO Secret Key
- MinIO Bucket Name
- NFS Storage Class Name
- LokiStack Size

### Using with inventory variables:

Add the following to your inventory file (`inventories/default/group_vars/all.yaml`):

```yaml
# Logging Stack Configuration
logging_stack:
  enabled: true
  minio_ip: "192.168.1.100"
  minio_access_key: "minioadmin"
  minio_secret_key: "minioadmin"
  minio_bucket: "loki-logs"
  nfs_storage_class: "nfs"
  loki_size: "1x.extra-small"
```

Then run:

```bash
ansible-playbook playbooks/8_deploy_logging_stack.yaml
```

## Prerequisites Setup

### 1. Install MinIO

You can use the provided MinIO setup script:

```bash
# On a separate server or the bastion
git clone https://github.ibm.com/Irappa-Pattar/OCP-Observability-Setup-Tool.git
cd OCP-Observability-Setup-Tool
chmod +x minio.sh
./minio.sh
```

After installation:
1. Login to MinIO console (http://minio-ip:9001)
2. Create a bucket (e.g., "loki-logs")
3. Generate access key and secret key

### 2. Setup NFS Storage

You can use the provided NFS setup script:

```bash
# On a separate server
git clone https://github.ibm.com/Irappa-Pattar/OCP-Observability-Setup-Tool.git
cd OCP-Observability-Setup-Tool
chmod +x nfs.sh
./nfs.sh
```

### 3. Install Python Kubernetes Module

On the bastion host:

```bash
pip3 install kubernetes
```

Or using Ansible:

```bash
ansible-playbook -i inventories/default playbooks/0_setup.yaml
```

## Post-Deployment

### Verify Installation

```bash
# Check operators
oc get csv -n openshift-logging

# Check LokiStack
oc get lokistack -n openshift-logging

# Check pods
oc get pods -n openshift-logging

# Check ClusterLogForwarder
oc get clusterlogforwarder -n openshift-logging

# Check UI Plugin
oc get uiplugin
```

### Access Logs

1. Login to OpenShift Console
2. Navigate to: **Observe** > **Logs**
3. Select namespace and pod to view logs
4. Use LogQL queries to filter logs

### Deploy Log Generator (Optional)

To test the logging stack:

```bash
oc apply -f roles/deploy_logging_stack/templates/log-generator.yaml.j2
```

Or use the role to deploy it:

```yaml
- name: Deploy log generator
  kubernetes.core.k8s:
    state: present
    src: "{{ role_path }}/templates/log-generator.yaml.j2"
```

## Troubleshooting

### Operators not installing

```bash
# Check operator subscriptions
oc get subscription -n openshift-logging
oc get subscription -n openshift-operators

# Check install plans
oc get installplan -n openshift-logging
oc get installplan -n openshift-operators
```

### LokiStack pods not starting

```bash
# Check LokiStack status
oc describe lokistack lokistack-sample -n openshift-logging

# Check PVC status
oc get pvc -n openshift-logging

# Check S3 secret
oc get secret credsecret -n openshift-logging -o yaml
```

### Logs not appearing

```bash
# Check ClusterLogForwarder status
oc get clusterlogforwarder collector -n openshift-logging -o yaml

# Check collector pods
oc get pods -n openshift-logging | grep collector

# Check collector logs
oc logs -n openshift-logging -l app.kubernetes.io/component=collector
```

### MinIO connection issues

```bash
# Test MinIO connectivity from bastion
curl http://<minio-ip>:9000

# Check MinIO credentials
oc get secret credsecret -n openshift-logging -o jsonpath='{.data.access_key_id}' | base64 -d
```

## License

Apache License 2.0

## Author Information

This role was created as part of the OpenShift logging stack deployment automation for SNO clusters.