# Velero Installation and Usage Guide

This guide provides a step-by-step Method of Procedure (MOP) for installing Velero on a Kubernetes cluster, and for performing backup and restore operations.

## 1. Prerequisites

Before you begin, ensure you have the following:

*   **`kubectl` access** to your Kubernetes cluster.
*   **An NFS provisioner** running in your cluster.
*   **The Velero command-line interface (CLI)** installed on your local machine.

## 2. Installation

This section covers the installation of Velero and its components.

### 2.1. Install the Velero CLI

If you haven't already, download the Velero CLI binary for your operating system from the [Velero releases page](https://github.com/vmware-tanzu/velero/releases) and place it in your `PATH`.

### 2.2. Deploy MinIO for S3-Compatible Storage

Velero requires an S3-compatible object storage backend. We will deploy MinIO to the cluster and configure it to use your NFS provisioner for storage.

1.  **Create the `velero` namespace:**

    ```bash
    kubectl create namespace velero
    ```

2.  **Create a file named `minio-on-nfs.yaml`** with the following content:

    ```yaml
    apiVersion: v1
    kind: Secret
    metadata:
      name: minio-creds
      namespace: velero
    type: Opaque
    stringData:
      accesskey: "minio"
      secretkey: "minio123"
    ---
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: minio-pvc
      namespace: velero
    spec:
      accessModes:
        - ReadWriteMany
      resources:
        requests:
          storage: 20Gi
      storageClassName: nfs-storage
    ---
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: minio
      namespace: velero
      labels:
        app: minio
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: minio
      template:
        metadata:
          labels:
            app: minio
        spec:
          containers:
          - name: minio
            image: minio/minio:latest
            args:
              - server
              - /data
              - --console-address
              - ":9001"
            env:
              - name: MINIO_ROOT_USER
                valueFrom:
                  secretKeyRef:
                    name: minio-creds
                    key: accesskey
              - name: MINIO_ROOT_PASSWORD
                valueFrom:
                  secretKeyRef:
                    name: minio-creds
                    key: secretkey
            ports:
              - containerPort: 9000
                name: s3
              - containerPort: 9001
                name: console
            volumeMounts:
              - name: data
                mountPath: /data
          volumes:
            - name: data
              persistentVolumeClaim:
                claimName: minio-pvc
    ---
    apiVersion: v1
    kind: Service
    metadata:
      name: minio
      namespace: velero
    spec:
      selector:
        app: minio
      ports:
        - name: s3
          port: 9000
          targetPort: 9000
        - name: console
          port: 9001
          targetPort: 9001
      type: ClusterIP
    ```

3.  **Apply the configuration:**

    ```bash
    kubectl apply -f minio-on-nfs.yaml
    ```

### 2.3. Install Velero

1.  **Create a credentials file** for Velero to access MinIO:

    ```bash
    echo "[default]
    aws_access_key_id = minio
    aws_secret_access_key = minio123" > credentials-velero
    ```

2.  **Install Velero** using the `velero install` command:

    ```bash
    velero install \
        --provider aws \
        --plugins velero/velero-plugin-for-aws:v1.7.0 \
        --bucket velero \
        --secret-file ./credentials-velero \
        --use-volume-snapshots=false \
        --backup-location-config region=minio,s3ForcePathStyle="true",s3Url=http://minio.velero.svc:9000
    ```

3.  **Verify the installation** by checking the status of the Velero deployment:

    ```bash
    kubectl get pods -n velero
    ```

    You should see the Velero and MinIO pods in the `Running` state.

## 3. Backup Operations

This section describes how to perform backup operations using Velero.

### 3.1. Back Up the Entire Cluster

To create a backup of all resources in the cluster, run the following command:

```bash
velero backup create <BACKUP_NAME>
```

Replace `<BACKUP_NAME>` with a descriptive name for your backup (e.g., `full-cluster-backup-20250915`).

### 3.2. Check Backup Status

To check the status of a backup, use the `velero backup describe` command:

```bash
velero backup describe <BACKUP_NAME>
```

The `Phase` field will indicate the status of the backup. A successful backup will show `Completed`.

### 3.3. Back Up a Specific Namespace

To back up only the resources in a specific namespace, use the `--include-namespaces` flag:

```bash
velero backup create <BACKUP_NAME> --include-namespaces <NAMESPACE>
```

## 4. Restore Operations

This section describes how to perform restore operations using Velero.

### 4.1. Restore the Entire Cluster

To restore the entire cluster from a backup, run the following command:

```bash
velero restore create --from-backup <BACKUP_NAME>
```

### 4.2. Check Restore Status

To check the status of a restore, use the `velero restore describe` command:

```bash
velero restore describe <RESTORE_NAME>
```

The `Phase` field will indicate the status of the restore. A successful restore will show `Completed`.

### 4.3. Restore a Specific Namespace

To restore only a specific namespace from a backup, use the `--include-namespaces` flag:

```bash
velero restore create --from-backup <BACKUP_NAME> --include-namespaces <NAMESPACE>
```

## 5. Troubleshooting

*   If a backup or restore fails, use the `velero backup logs` or `velero restore logs` commands to view the logs for the operation.
*   If you encounter issues with the MinIO deployment, check the logs of the MinIO pod for errors.
*   If you have issues with the NFS provisioner, check the logs of the `nfs-client-provisioner` pod.
