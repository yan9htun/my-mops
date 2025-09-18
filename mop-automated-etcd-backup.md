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

### 3. Step-by-Step Procedure

### Step 1: Create the Backup Script

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

### Step 2: Make the Script Executable

Make the script runnable by the system.

```bash
sudo chmod +x /usr/local/bin/etcd-backup.sh
```

### Step 3: Manually Test the Script

Before scheduling the script, run it manually to ensure it works correctly.

```bash
sudo /usr/local/bin/etcd-backup.sh
```

Check the output for any errors. Then, verify that a new `.db` file has been created in the `/var/lib/etcd-backups` directory.

```bash
ls -l /var/lib/etcd-backups/
```

### Step 4: Schedule the Cron Job

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

## 4. Security Considerations

*   **Secure Backup Files:** The script stores backups locally. You **must** add a step to your process to copy these snapshot files to a secure, remote location (e.g., an S3 bucket, a secure file server). This could be a separate cron job or a step added to the script itself.
*   **Permissions:** The backup directory and script should have restrictive permissions, typically only accessible by the `root` user.

---

## 5. Rollback

To disable the automated backup:

1.  **Open the root crontab:** `sudo crontab -e`
2.  **Delete the line** you added in Step 4.
3.  (Optional) Delete the script file: `sudo rm /usr/local/bin/etcd-backup.sh`
