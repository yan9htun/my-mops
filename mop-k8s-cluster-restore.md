# MOP: Kubernetes Cluster Restore from Snapshot

> **Purpose**: This document provides a focused, step-by-step Method of Procedure (MOP) for restoring a Kubernetes cluster and its applications from a complete disaster, using pre-existing `etcd` snapshots and Velero backups.

---

## 1. Disaster Scenario

*   **The Event:** The primary Kubernetes cluster, `k8s-prod-01`, has suffered a catastrophic, unrecoverable failure of its control-plane nodes.
*   **The Impact:** All applications are down. The `etcd` database is lost.
*   **The Assets:** We have the following assets available in a secure, off-site storage location:
    1.  A recent `etcd` database snapshot (e.g., `etcd-snapshot-YYYY-MM-DD_HH-MM-SS.db`).
    2.  A backup of the cluster's PKI directory (e.g., `pki-backup-YYYY-MM-DD_HH-MM-SS.tar.gz`).
    3.  Access to the Velero object storage bucket (MinIO), which contains application and PV backups.
*   **The Goal:** Provision a new cluster and restore it to the last known state using these assets.

---

## 2. Prerequisites

*   A new set of virtual or physical machines for the control-plane and worker nodes have been provisioned.
*   A fresh, clean installation of the same Kubernetes version as the failed cluster is running.
*   You have SSH access to the new control-plane nodes.
*   The `etcd` snapshot and PKI backup files have been securely copied to the primary master node (e.g., into `/tmp/dr-assets/`).

---

## Phase 1: Restore the Control Plane

**Objective:** Rebuild the cluster's brain (ETCD) and re-establish the control plane's identity (PKI).

### Step 1.1: Stop the New Control Plane

On the primary master node, we must stop the new, empty control plane to prevent it from interfering with the restore process.

```bash
# Create a temporary directory to hold the manifests
sudo mkdir -p /etc/kubernetes/manifests-stopped

# Move the manifests to stop the components
sudo mv /etc/kubernetes/manifests/kube-apiserver.yaml /etc/kubernetes/manifests-stopped/
sudo mv /etc/kubernetes/manifests/kube-controller-manager.yaml /etc/kubernetes/manifests-stopped/
sudo mv /etc/kubernetes/manifests/kube-scheduler.yaml /etc/kubernetes/manifests-stopped/
```
*Wait about a minute for the kubelet to stop the pods.*

### Step 1.2: Restore the PKI Certificates

Delete the new, clean PKI directory and replace it with the backed-up certificates.

```bash
# Remove the new PKI directory
sudo rm -rf /etc/kubernetes/pki

# Extract your backed-up PKI directory into place
sudo tar -xzvf /tmp/dr-assets/pki-backup-YYYY-MM-DD_HH-MM-SS.tar.gz -C /
```

### Step 1.3: Restore the ETCD Snapshot

Restore the `etcd` database from your snapshot into a **new** directory.

```bash
# Restore the snapshot to a new directory
sudo ETCDCTL_API=3 etcdctl snapshot restore /tmp/dr-assets/etcd-snapshot-YYYY-MM-DD_HH-M-SS.db \
  --data-dir /var/lib/etcd-from-restore
```

### Step 1.4: Update the ETCD Pod Manifest

Modify the main `etcd.yaml` manifest to point to the newly restored data directory.

1.  **Open the manifest for editing:**
    ```bash
    sudo nano /etc/kubernetes/manifests/etcd.yaml
    ```
2.  **Find the `etcd-data` volume:**
    Locate the section that looks like this:
    ```yaml
      - name: etcd-data
        hostPath:
          path: /var/lib/etcd
          type: DirectoryOrCreate
    ```
3.  **Change the `path`:**
    Modify the `path` to point to your new restore directory:
    ```yaml
      - name: etcd-data
        hostPath:
          path: /var/lib/etcd-from-restore
          type: DirectoryOrCreate
    ```
4.  **Save and close the file.**

### Step 1.5: Restart the Control Plane

Move the manifests back into place. The kubelet will detect them and start the control plane pods using the restored `etcd` data and `pki` certificates.

```bash
sudo mv /etc/kubernetes/manifests-stopped/* /etc/kubernetes/manifests/
```

### Step 1.6: Verify Control Plane Health

Wait several minutes for the system to stabilize. Then, check the status of the control plane.

```bash
# Check that the core components are running
kubectl get pods -n kube-system

# Check that nodes are beginning to join the cluster
kubectl get nodes
```
*You should see the `etcd`, `kube-apiserver`, `kube-controller-manager`, and `kube-scheduler` pods running. It may take time for all worker nodes to check in and become `Ready`.*

---

## Phase 2: Restore Applications and Data

**Objective:** Use Velero to restore all user-facing applications, deployments, and their persistent data.

### Step 2.1: Re-install Velero

Install Velero with the **exact same configuration** used in the old cluster. This allows it to connect to the MinIO bucket and see the existing backups.

```bash
# Use the same credentials file as before
cat > ./credentials-velero <<'EOF'
[default]
aws_access_key_id = minio
aws_secret_access_key = minio123
EOF

# Run the same install command
velero install \
  --provider aws \
  --plugins velero/velero-plugin-for-aws:v1.12.0 \
  --bucket velero \
  --secret-file ./credentials-velero \
  --backup-location-config region=minio,s3ForcePathStyle="true",s3Url=http://minio.velero.svc:9000 \
  --use-node-agent \
  --wait
```
*Note: You must also redeploy MinIO if it was running in the cluster, or ensure the external MinIO is reachable.*

### Step 2.2: Verify Backup Availability

Check that Velero can connect to the object storage and see the backups from the old cluster.

```bash
velero backup get
```
*You should see a list of your previously created backups (e.g., `daily-backup-...`).*

### Step 2.3: Initiate the Full Restore

Choose the latest successful backup and create a restore from it.

```bash
# Replace <LATEST_BACKUP_NAME> with the name of your most recent backup
velero restore create full-cluster-restore --from-backup <LATEST_BACKUP_NAME> --wait
```

### Step 2.4: Monitor and Verify the Restore

Check the status of the restore and the state of your applications.

```bash
# Describe the restore to see status and any errors
velero restore describe full-cluster-restore

# Check that application namespaces and pods are being created
kubectl get pods --all-namespaces

# Check PersistentVolumeClaims
kubectl get pvc --all-namespaces
```

If you find pods are stuck in `Pending` and PVCs are also `Pending`, you may need to patch the `Released` Persistent Volumes as described in the Velero MOP.

---

## 3. Conclusion

At this point, the control plane has been restored from the `etcd` snapshot, and all applications and their data have been restored by Velero. The cluster is now back to its last known good state. It is critical to perform a full functional check of all key applications.

```
