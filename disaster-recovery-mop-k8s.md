# MOP: Kubernetes Cluster Disaster Recovery

> **Purpose**: This document provides a Method of Procedure (MOP) for full Kubernetes cluster disaster recovery. It covers the three critical components: ETCD database backup, application/workload backup, and PKI certificate backup.

**IMPORTANT:** This is an advanced procedure. Performing these steps on a live production cluster can cause irreversible damage if done incorrectly. Always test your DR plan in a non-production environment first.

---

## DR Strategy Overview

A complete cluster recovery depends on restoring three distinct types of data:

1.  **ETCD Database**: The brain of your cluster. It stores the state of all Kubernetes resources. We will use `etcdctl snapshot` for this.
2.  **Application Data (Persistent Volumes)**: The actual data used by your stateful applications. We will use **Velero with Kopia** for this.
3.  **PKI Certificates**: The TLS certificates used by cluster components to communicate securely. We will back these up manually.

This guide is structured to address these components separately for backup and restore.

---

## Part A: Control Plane Backup

This part must be performed on a master/control-plane node.

### Section 1: ETCD Database Snapshot

This is the most critical backup. It must be performed regularly.

1.  **Set the API Version:**

    ```bash
    export ETCDCTL_API=3
    ```

2.  **Execute the Snapshot Command:**
    This command connects to the local etcd server using its certificates and creates a snapshot.

    ```bash
    sudo ETCDCTL_API=3 etcdctl snapshot save /var/lib/etcd-backups/etcd-snapshot-$(date +%Y-%m-%d_%H-%M-%S).db \
      --cacert /etc/kubernetes/pki/etcd/ca.crt \
      --cert /etc/kubernetes/pki/etcd/server.crt \
      --key /etc/kubernetes/pki/etcd/server.key \
      --endpoints https://127.0.0.1:2379
    ```
    *Ensure the backup directory (`/var/lib/etcd-backups/`) exists and is secure.*

3.  **Verify the Snapshot:**

    ```bash
    sudo ETCDCTL_API=3 etcdctl snapshot status --write-out=table /var/lib/etcd-backups/<your-snapshot-file.db>
    ```

4.  **Secure the Snapshot:**
    Move the snapshot file to a secure, off-site location.

### Section 2: PKI Certificate Backup

Backup the entire Public Key Infrastructure directory.

```bash
sudo tar -czvf /var/lib/etcd-backups/pki-backup-$(date +%Y-%m-%d_%H-%M-%S).tar.gz /etc/kubernetes/pki
```
Secure this archive along with your etcd snapshot.

---

## Part B: Application and Data Backup (Velero)

This part covers the backup of all Kubernetes objects (Deployments, Services, etc.) and the data in their Persistent Volumes. We will use Velero with MinIO as the storage backend.

*(This section is adapted from the Velero v1.16.2 MOP)*

### 1. Install Velero CLI (v1.16.2)

Ensure the Velero CLI is installed on your management machine.

```bash
velero version
# Should be v1.16.2 or compatible
```

### 2. Deploy MinIO for Velero Storage

Create the `velero` namespace and a credentials file.

```bash
kubectl create namespace velero || true
cat > ./credentials-velero <<'EOF'
[default]
aws_access_key_id = minio
aws_secret_access_key = minio123
EOF
```

Deploy MinIO using the `minio-on-nfs.yaml` manifest.

```bash
kubectl apply -f minio-on-nfs.yaml
kubectl -n velero get pods -l app=minio -w
```

### 3. Create the `velero` Bucket

Use a temporary pod to create the bucket inside MinIO.

```bash
kubectl -n velero run mc --image=minio/mc --restart=Never --command -- sleep 3600
sleep 15
kubectl -n velero exec pod/mc -- mc alias set myminio http://minio:9000 minio minio123 --api S3v4
kubectl -n velero exec pod/mc -- mc mb myminio/velero || true
kubectl -n velero delete pod/mc --now
```

### 4. Install Velero Server

Install Velero, configuring it to use our MinIO bucket and Kopia for volume backups.

```bash
velero install \
  --provider aws \
  --plugins velero/velero-plugin-for-aws:v1.12.0 \
  --bucket velero \
  --secret-file ./credentials-velero \
  --backup-location-config region=minio,s3ForcePathStyle="true",s3Url=http://minio.velero.svc:9000 \
  --use-node-agent \
  --wait
```

### 5. Schedule Regular Backups

For a DR plan, you must have automated, recurring backups.

```bash
# Create a backup of all non-system namespaces every 6 hours
velero schedule create daily-backup \
  --schedule="0 */6 * * *" \
  --exclude-namespaces kube-system,velero
```

Check the status of your scheduled backup:
`velero backup get`

---

## Part C: Full Restore Procedure

**WARNING:** This is a highly destructive process. The cluster will be down. Only proceed if you are in a true disaster scenario.

### Section 1: Restore the Control Plane

1.  **Prepare the Master Node:** Copy your `etcd` snapshot and `pki` backup to the master node of your new/rebuilt cluster.

2.  **Stop the Control Plane:** Stop the kube-apiserver and other control plane components. The easiest way is to temporarily move their manifests.

    ```bash
    sudo mkdir /etc/kubernetes/manifests-stopped
    sudo mv /etc/kubernetes/manifests/kube-apiserver.yaml /etc/kubernetes/manifests-stopped/
    sudo mv /etc/kubernetes/manifests/kube-controller-manager.yaml /etc/kubernetes/manifests-stopped/
    sudo mv /etc/kubernetes/manifests/kube-scheduler.yaml /etc/kubernetes/manifests-stopped/
    ```

3.  **Restore ETCD Snapshot:**
    Restore the snapshot to a **new** data directory.

    ```bash
    sudo ETCDCTL_API=3 etcdctl snapshot restore /path/to/your/etcd-snapshot.db \
      --data-dir /var/lib/etcd-from-restore
    ```

4.  **Update the ETCD Manifest:**
    Edit the main etcd manifest (`/etc/kubernetes/manifests/etcd.yaml`) and change the `hostPath` for the `etcd-data` volume to point to your new restore directory (`/var/lib/etcd-from-restore`).

5.  **Restore PKI:**
    Restore the PKI directory.

    ```bash
    sudo rm -rf /etc/kubernetes/pki
    sudo tar -xzvf /path/to/your/pki-backup.tar.gz -C /
    ```

6.  **Restart the Control Plane:**
    Move the manifests back. The kubelet will detect the changes and restart the control plane components using the restored data.

    ```bash
    sudo mv /etc/kubernetes/manifests-stopped/* /etc/kubernetes/manifests/
    ```
    Monitor the control plane pods in `kube-system` until they are healthy.

### Section 2: Restore Applications with Velero

Once the control plane is stable, reinstall Velero with the *exact same configuration* as before (Part B, step 4). Velero will see the existing backups in the MinIO bucket.

1.  **Verify Velero can see the backups:**

    ```bash
    velero backup get
    ```
    You should see the backups created by your schedule.

2.  **Initiate the Full Restore:**
    Restore all content from your latest backup.

    ```bash
    velero restore create --from-backup <name-of-your-latest-backup> --wait
    ```

3.  **Verify Application State:**
    Check that your applications and their PVCs are restored and running correctly, using the troubleshooting steps from the previous guide if needed.
