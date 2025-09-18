# MOP: Automated ETCD Snapshot Backup

> **Purpose**: This document provides a Method of Procedure (MOP) for creating and scheduling a script to automatically back up the Kubernetes ETCD database every 4 hours. This script should be deployed on all control-plane nodes.

---

## 1. Objective

To automate the process of creating timestamped ETCD snapshots and to periodically clean up old backups to prevent disk exhaustion.

This procedure uses a standard cron job on the control-plane host, as this is the most reliable method for accessing `etcdctl` and the required host paths.

---

## 2. Prerequisites

*   You must have SSH access to your Kubernetes control-plane (master) nodes.
*   The `etcdctl` utility must be installed on the master nodes.
*   This procedure should be configured on **each** master node to ensure backups are taken even if one node is down.

---

## 3. Step-by-Step Backup Procedure

### Step 3.1: Create the Backup Script

First, we will create the shell script that performs the backup and cleanup operations.

1.  **SSH into a master node.**

2.  **Create the script file:**
    Create a new file, for example, at `/usr/local/bin/etcd-backup.sh`.

    ```bash
    sudo nano /usr/local/bin/etcd-backup.sh
    ```

3.  **Add the following content to the script:**

    ```sh
    #!/bin/bash

    # Exit immediately if a command exits with a non-zero status.
    set -e

    # --- Configuration ---
    export ETCDCTL_API=3
    BACKUP_DIR="/var/lib/etcd-backups"
    DAYS_TO_KEEP=7 # Number of days to keep old backups
    ETCD_CERT="/etc/kubernetes/pki/etcd/server.crt"
    ETCD_CACERT="/etc/kubernetes/pki/etcd/ca.crt"
    ETCD_KEY="/etc/kubernetes/pki/etcd/server.key"
    ENDPOINTS="https://127.0.0.1:2379"

    # --- Main Logic ---

    # 1. Create backup directory if it doesn't exist
    mkdir -p "${BACKUP_DIR}"

    # 2. Create the timestamped snapshot
    SNAPSHOT_FILE="${BACKUP_DIR}/etcd-snapshot-$(date +%Y-%m-%d_%H-%M-%S).db"
    echo "Creating ETCD snapshot: ${SNAPSHOT_FILE}"
    etcdctl snapshot save "${SNAPSHOT_FILE}" \
      --cacert="${ETCD_CACERT}" \
      --cert="${ETCD_CERT}" \
      --key="${ETCD_KEY}" \
      --endpoints="${ENDPOINTS}"

    echo "Snapshot created successfully."

    # 3. Verify the snapshot
    echo "Verifying snapshot..."
    etcdctl snapshot status --write-out=table "${SNAPSHOT_FILE}"

    # 4. Clean up old backups
    echo "Cleaning up backups older than ${DAYS_TO_KEEP} days..."
    find "${BACKUP_DIR}" -name "*.db" -mtime +${DAYS_TO_KEEP} -exec echo "Deleting: {}" \; -exec rm {} \;

    echo "ETCD backup process complete."
    ```

4.  **Save and exit the editor.**

### Step 3.2: Make the Script Executable

Make the script runnable by the system.

```bash
sudo chmod +x /usr/local/bin/etcd-backup.sh
```

### Step 3.3: Manually Test the Script

Before scheduling the script, run it manually to ensure it works correctly.

```bash
sudo /usr/local/bin/etcd-backup.sh
```

Check the output for any errors. Then, verify that a new `.db` file has been created in the `/var/lib/etcd-backups` directory.

```bash
ls -l /var/lib/etcd-backups/
```

### Step 3.4: Schedule the Cron Job

Now, we will use `cron` to schedule the script to run every 4 hours.

1.  **Open the root user's crontab:**

    ```bash
    sudo crontab -e
    ```

2.  **Add the following line to the end of the file:**
    This schedules the script to run at the top of the hour, every 4 hours (e.g., at 00:00, 04:00, 08:00, etc.).

    ```
    0 */4 * * * /usr/local/bin/etcd-backup.sh > /var/log/etcd-backup.log 2>&1
    ```
    *Redirecting the output (`> ... 2>&1`) is a best practice that helps you debug the cron job if it fails silently.*

3.  **Save and exit the editor.** The cron service will automatically pick up the new schedule.

---

## 4. Manual ETCD Restore Procedure

> **WARNING:** This is a highly destructive procedure for disaster recovery. Only perform this on a new cluster or after a catastrophic failure of the existing control plane. Incorrectly performing these steps **will break your cluster**.

### Scenario

*   **The Event:** The cluster's control plane is non-functional and must be restored from a snapshot.
*   **The Assets:** You have a trusted `etcd` snapshot file available on the master node.

### Step 4.1: Stop the Kubelet Service

On the primary master node, stop the kubelet service. This will stop all control plane pods, preventing any interference with the restore process.

```bash
sudo systemctl stop kubelet.service
```

### Step 4.2: Restore the ETCD Snapshot

This step restores the database from your snapshot file into a temporary directory.

```bash
# --- Configuration ---
# !!! IMPORTANT: Set this to the full path of the snapshot you want to restore !!!
SNAPSHOT_TO_RESTORE="/var/lib/etcd-backups/etcd-snapshot-YYYY-MM-DD_HH-MM-SS.db"

RESTORE_DIR="/var/lib/etcd-from-restore"
ETCD_CERT="/etc/kubernetes/pki/etcd/server.crt"
ETCD_CACERT="/etc/kubernetes/pki/etcd/ca.crt"
ETCD_KEY="/etc/kubernetes/pki/etcd/server.key"

# --- Restore Logic ---
echo "Restoring ETCD from snapshot: ${SNAPSHOT_TO_RESTORE}"

# Restore the snapshot to the new, temporary directory
sudo ETCDCTL_API=3 etcdctl snapshot restore "${SNAPSHOT_TO_RESTORE}" \
  --data-dir "${RESTORE_DIR}" \
  --cacert "${ETCD_CACERT}" \
  --cert "${ETCD_CERT}" \
  --key "${ETCD_KEY}"

echo "Restore complete."
```


### Step 4.3: Replace the ETCD Data Directory

Move the old, empty or corrupted `etcd` data directory aside and replace it with the directory containing your restored data.

```bash
# Move the old directory with a timestamped name
sudo mv /var/lib/etcd /var/lib/etcd-old-$(date +%s)

# Move the restored directory into the correct place
sudo mv /var/lib/etcd-from-restore /var/lib/etcd
```

### Step 4.4: Restart the Kubelet Service

Now that the restored data is in the correct location, restart the kubelet. It will start the control plane pods, which will now use the restored data.

```bash
sudo systemctl start kubelet.service
```

### Step 4.5: Verify Control Plane Health

Wait several minutes for the system to stabilize. The kubelet needs time to start the control plane pods. Then, check the status from a machine with `kubectl` access.

```bash
# Check that the core components are running
kubectl get pods -n kube-system

# Check the status of the nodes
kubectl get nodes
```
*You should see the `etcd`, `kube-apiserver`, etc. pods running. It may take time for all nodes to report a `Ready` status.*
