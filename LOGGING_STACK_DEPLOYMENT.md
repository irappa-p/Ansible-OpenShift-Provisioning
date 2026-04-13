# OpenShift Logging Stack Deployment Guide

This guide explains how to deploy a complete logging stack on your SNO (Single Node OpenShift) cluster using the Ansible-OpenShift-Provisioning repository.

## Overview

The logging stack includes:
- **Loki Operator**: Log aggregation system
- **Cluster Logging Operator**: Manages log collection
- **Local Storage Operator**: Provides local storage capabilities
- **LokiStack**: Loki deployment with S3 backend (MinIO)
- **ClusterLogForwarder**: Forwards logs to Loki
- **Cluster Observability Operator**: Enables observability features
- **Logging UI Plugin**: Web UI for viewing logs in OpenShift Console

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    OpenShift Console                         │
│                  (Observe > Logs UI)                         │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│              Cluster Observability Operator                  │
│                   (UI Plugin Manager)                        │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                      LokiStack                               │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │  Distributor │  │   Ingester   │  │   Querier    │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
│         │                  │                  │              │
│         └──────────────────┴──────────────────┘              │
│                            │                                 │
│                            ▼                                 │
│                   ┌─────────────────┐                        │
│                   │  MinIO (S3)     │                        │
│                   │  Object Storage │                        │
│                   └─────────────────┘                        │
└─────────────────────────────────────────────────────────────┘
                              ▲
                              │
┌─────────────────────────────────────────────────────────────┐
│              ClusterLogForwarder                             │
│         (Log Collection & Forwarding)                        │
└─────────────────────────────────────────────────────────────┘
                              ▲
                              │
┌─────────────────────────────────────────────────────────────┐
│         Application, Infrastructure, Audit Logs             │
│              (From all cluster components)                   │
└─────────────────────────────────────────────────────────────┘
```

## Prerequisites

### 1. SNO Cluster Deployed

Ensure your SNO cluster is successfully deployed using the main playbook:

```bash
cd /path/to/Ansible-OpenShift-Provisioning
ansible-playbook playbooks/master_playbook_for_abi.yaml
```

### 2. MinIO Server Setup

MinIO provides S3-compatible object storage for Loki.

#### Option A: Using the provided script

```bash
# Clone the observability setup tool
git clone https://github.ibm.com/Irappa-Pattar/OCP-Observability-Setup-Tool.git
cd OCP-Observability-Setup-Tool

# Run MinIO setup script
chmod +x minio.sh
./minio.sh
```

#### Option B: Manual MinIO setup

```bash
# Install MinIO
wget https://dl.min.io/server/minio/release/linux-amd64/minio
chmod +x minio
sudo mv minio /usr/local/bin/

# Create data directory
sudo mkdir -p /data/minio

# Start MinIO
minio server /data/minio --console-address ":9001"
```

#### Post-installation steps:

1. Access MinIO Console: `http://<minio-server-ip>:9001`
2. Login with default credentials (minioadmin/minioadmin)
3. Create a bucket (e.g., "loki-logs")
4. Generate Access Key and Secret Key:
   - Navigate to: Identity > Service Accounts
   - Click "Create Service Account"
   - Save the Access Key and Secret Key

### 3. NFS Server Setup

NFS provides persistent storage for Loki components.

#### Option A: Using the provided script

```bash
# Clone the observability setup tool
git clone https://github.ibm.com/Irappa-Pattar/OCP-Observability-Setup-Tool.git
cd OCP-Observability-Setup-Tool

# Run NFS setup script
chmod +x nfs.sh
./nfs.sh
```

#### Option B: Manual NFS setup

```bash
# Install NFS server
sudo yum install -y nfs-utils

# Create export directory
sudo mkdir -p /exports/loki

# Configure NFS exports
echo "/exports/loki *(rw,sync,no_root_squash,no_subtree_check)" | sudo tee -a /etc/exports

# Start NFS services
sudo systemctl enable --now nfs-server
sudo exportfs -ra
```

### 4. Install Python Kubernetes Module

On the bastion host:

```bash
pip3 install kubernetes
```

Or install via Ansible:

```bash
ansible-playbook -i inventories/default playbooks/0_setup.yaml
```

### 5. Login to OpenShift Cluster

From the bastion host:

```bash
# Get kubeadmin password
cat ~/auth/kubeadmin-password

# Login to cluster
oc login -u kubeadmin -p <password> https://api.<cluster-name>.<base-domain>:6443
```

## Deployment Methods

### Method 1: Interactive Deployment (Recommended)

This method prompts you for all required information:

```bash
cd /path/to/Ansible-OpenShift-Provisioning
ansible-playbook playbooks/deploy_logging_stack_master.yaml
```

You will be prompted for:
- MinIO Server IP
- MinIO Access Key
- MinIO Secret Key
- MinIO Bucket Name (default: test)
- NFS Storage Class Name (default: nfs)
- LokiStack Size (default: 1x.extra-small)

### Method 2: Using Inventory Variables

1. Edit your inventory file:

```bash
vi inventories/default/group_vars/all.yaml
```

2. Add logging stack configuration:

```yaml
# Logging Stack Configuration
logging_stack:
  enabled: true
  minio_ip: "192.168.1.100"
  minio_access_key: "your-access-key"
  minio_secret_key: "your-secret-key"
  minio_bucket: "loki-logs"
  nfs_storage_class: "nfs"
  loki_size: "1x.extra-small"
```

3. Run the playbook:

```bash
ansible-playbook playbooks/8_deploy_logging_stack.yaml
```

### Method 3: Command Line Variables

```bash
ansible-playbook playbooks/8_deploy_logging_stack.yaml \
  -e "minio_ip=192.168.1.100" \
  -e "minio_access_key=your-access-key" \
  -e "minio_secret_key=your-secret-key" \
  -e "minio_bucket=loki-logs" \
  -e "nfs_storage_class=nfs" \
  -e "loki_size=1x.extra-small"
```

## LokiStack Sizing

Choose the appropriate size based on your log volume:

| Size | Description | Use Case |
|------|-------------|----------|
| `1x.extra-small` | Minimal resources | Testing, small deployments |
| `1x.small` | Small production | Low log volume |
| `1x.medium` | Medium production | Moderate log volume |

## Verification

### 1. Check Operator Installation

```bash
# Check operators in openshift-logging namespace
oc get csv -n openshift-logging

# Expected output:
# NAME                                         DISPLAY                     VERSION   REPLACES   PHASE
# cluster-logging.v6.x.x                      Red Hat OpenShift Logging   6.x.x                Succeeded
# loki-operator.v6.x.x                        Loki Operator               6.x.x                Succeeded

# Check operators in openshift-operators namespace
oc get csv -n openshift-operators | grep -E "local-storage|cluster-observability"
```

### 2. Check LokiStack Status

```bash
# Check LokiStack resource
oc get lokistack -n openshift-logging

# Expected output:
# NAME               AGE
# lokistack-sample   5m

# Check detailed status
oc describe lokistack lokistack-sample -n openshift-logging
```

### 3. Check Pods

```bash
# Check all pods in openshift-logging namespace
oc get pods -n openshift-logging

# Expected pods:
# - lokistack-sample-compactor-*
# - lokistack-sample-distributor-*
# - lokistack-sample-gateway-*
# - lokistack-sample-index-gateway-*
# - lokistack-sample-ingester-*
# - lokistack-sample-querier-*
# - lokistack-sample-query-frontend-*
# - collector-*
```

### 4. Check ClusterLogForwarder

```bash
oc get clusterlogforwarder -n openshift-logging

# Check status
oc describe clusterlogforwarder collector -n openshift-logging
```

### 5. Check UI Plugin

```bash
oc get uiplugin

# Expected output:
# NAME      AGE
# logging   5m
```

## Accessing Logs

### Via OpenShift Console

1. Login to OpenShift Console: `https://console-openshift-console.apps.<cluster-name>.<base-domain>`
2. Navigate to: **Observe** > **Logs**
3. Select:
   - **Namespace**: Choose the namespace
   - **Pod**: Select a pod
   - **Container**: Select a container (if multiple)
4. View logs in real-time

### Using LogQL Queries

LogQL is Loki's query language. Examples:

```logql
# All logs from a namespace
{kubernetes_namespace_name="openshift-logging"}

# Logs with specific label
{app="my-app"}

# Filter by log level
{kubernetes_namespace_name="default"} |= "ERROR"

# Count errors in last hour
count_over_time({kubernetes_namespace_name="default"} |= "ERROR" [1h])
```

## Testing the Logging Stack

### Deploy Log Generator

```bash
# Apply the log generator
oc apply -f roles/deploy_logging_stack/templates/log-generator.yaml.j2

# Check if it's running
oc get pods -n log-generator

# View logs being generated
oc logs -f -n log-generator deployment/log-generator
```

### View Generated Logs in Console

1. Go to **Observe** > **Logs**
2. Select namespace: `log-generator`
3. Select pod: `log-generator-*`
4. You should see logs with different levels (INFO, DEBUG, WARN, ERROR)

## Troubleshooting

### Issue: Operators not installing

**Symptoms**: CSV shows "Installing" or "Failed" status

**Solution**:
```bash
# Check subscription
oc get subscription -n openshift-logging

# Check install plan
oc get installplan -n openshift-logging

# Check operator pod logs
oc logs -n openshift-logging -l app=loki-operator
```

### Issue: LokiStack pods not starting

**Symptoms**: Pods in CrashLoopBackOff or Pending state

**Solution**:
```bash
# Check LokiStack status
oc describe lokistack lokistack-sample -n openshift-logging

# Check PVC status
oc get pvc -n openshift-logging

# Check events
oc get events -n openshift-logging --sort-by='.lastTimestamp'

# Verify NFS storage is accessible
showmount -e <nfs-server-ip>
```

### Issue: Cannot connect to MinIO

**Symptoms**: Loki pods show S3 connection errors

**Solution**:
```bash
# Test MinIO connectivity from bastion
curl http://<minio-ip>:9000

# Verify secret
oc get secret credsecret -n openshift-logging -o yaml

# Check MinIO credentials
oc get secret credsecret -n openshift-logging -o jsonpath='{.data.access_key_id}' | base64 -d
oc get secret credsecret -n openshift-logging -o jsonpath='{.data.access_key_secret}' | base64 -d

# Test S3 access from a pod
oc run -it --rm debug --image=amazon/aws-cli --restart=Never -- \
  s3 ls --endpoint-url http://<minio-ip>:9000 \
  --access-key <access-key> \
  --secret-key <secret-key>
```

### Issue: Logs not appearing in UI

**Symptoms**: UI shows no logs or "No datapoints found"

**Solution**:
```bash
# Check ClusterLogForwarder status
oc get clusterlogforwarder collector -n openshift-logging -o yaml

# Check collector pods
oc get pods -n openshift-logging | grep collector

# Check collector logs
oc logs -n openshift-logging -l app.kubernetes.io/component=collector

# Verify log generator is running
oc get pods -n log-generator
oc logs -n log-generator deployment/log-generator
```

### Issue: UI Plugin not showing

**Symptoms**: "Logs" option not visible in Observe menu

**Solution**:
```bash
# Check UI Plugin
oc get uiplugin

# Check Cluster Observability Operator
oc get csv -n openshift-operators | grep cluster-observability

# Restart console pods
oc delete pods -n openshift-console -l app=console
```

## Cleanup

To remove the logging stack:

```bash
# Delete ClusterLogForwarder
oc delete clusterlogforwarder collector -n openshift-logging

# Delete LokiStack
oc delete lokistack lokistack-sample -n openshift-logging

# Delete UI Plugin
oc delete uiplugin logging

# Uninstall operators (optional)
oc delete subscription cluster-logging -n openshift-logging
oc delete subscription loki-operator -n openshift-logging
oc delete subscription cluster-observability-operator -n openshift-operators
oc delete subscription local-storage-operator -n openshift-operators

# Delete namespace (optional)
oc delete namespace openshift-logging
```

## Additional Resources

- [OpenShift Logging Documentation](https://docs.openshift.com/container-platform/latest/logging/cluster-logging.html)
- [Loki Documentation](https://grafana.com/docs/loki/latest/)
- [LogQL Query Language](https://grafana.com/docs/loki/latest/logql/)
- [MinIO Documentation](https://min.io/docs/minio/linux/index.html)

## Support

For issues or questions:
1. Check the troubleshooting section above
2. Review OpenShift and Loki documentation
3. Check operator logs for detailed error messages
4. Verify all prerequisites are met

## License

Apache License 2.0