# Add New Storage with LVM and XFS on RHEL

This guide provides a step-by-step procedure for adding and configuring a new storage device on Red Hat Enterprise Linux (RHEL) using Logical Volume Management (LVM) and the XFS file system.

---

### **1. Partition the New Disk**

First, verify the new disk is visible to the system and start the `fdisk` utility. In this example, we assume the new disk is `/dev/sdb`.

1.  **List block devices** to identify your new disk.

    ```bash
    lsblk
    ```

2.  **Start `fdisk`** for the new disk.

    ```bash
    sudo fdisk /dev/sdb
    ```

#### **Create a New Partition**

-   Type `n` to create a new partition.
-   Choose `p` for a primary partition.
-   Accept the default partition number (usually `1`).
-   Accept the default first sector (press Enter).
-   Accept the default last sector to use the entire disk (press Enter).

#### **Set the Partition Type**

-   Type `t` to change the partition type.
-   Enter the partition number (usually `1`).
-   Type `8e` to set the type to **Linux LVM**.

#### **Write Changes**

-   Type `w` to write the changes to the disk and exit `fdisk`.

---

### **2. Create LVM Components**

Now, create a physical volume, a volume group, and a logical volume using the newly created partition.

1.  **Create a physical volume** on the new partition (`/dev/sdb1`).

    ```bash
    sudo pvcreate /dev/sdb1
    ```

2.  **Create a volume group** named `rhel-data` using the physical volume.

    ```bash
    sudo vgcreate rhel-data /dev/sdb1
    ```

3.  **Create a logical volume** named `data` that uses all the free space in the `rhel-data` volume group.

    ```bash
    sudo lvcreate -n data -l 100%FREE rhel-data
    ```

---

### **3. Format and Mount the Volume**

Next, you will format the logical volume with a file system and mount it.

1.  **Format the logical volume** with the XFS file system.

    ```bash
    sudo mkfs.xfs /dev/rhel-data/data
    ```

2.  **Create a mount point** directory.

    ```bash
    sudo mkdir /data
    ```

3.  **Mount the logical volume** to the new directory.

    ```bash
    sudo mount /dev/rhel-data/data /data
    ```

4.  **Verify the mount** and check the available space.

    ```bash
    df -h /data
    ```

---

### **4. Configure for Permanent Mount**

To ensure the new volume is automatically mounted on system reboot, add an entry to the `/etc/fstab` file.

1.  **Edit the `fstab` file** using a text editor like `vi`.

    ```bash
    sudo vi /etc/fstab
    ```

2.  **Add the following line** to the end of the file.

    ```
    /dev/rhel-data/data     /data     xfs     defaults     0     2
    ```

---

You have successfully added and configured a new storage device using LVM and XFS.
