# Red Hat Enterprise Linux (RHEL) Upgrade with Leapp

This guide provides a detailed procedure for upgrading a RHEL 7 system to RHEL 8 using the Leapp utility. It includes pre-upgrade checks, subscription management, and the final upgrade process.

---

### **1. Pre-Upgrade Checks**

Before starting the upgrade, perform these checks to ensure your system is ready.

* **Check Network Connectivity**: Use `curl` to confirm the system can access the internet. This is crucial for downloading packages.

    ```bash
    curl [www.google.com](https://www.google.com)
    ```

* **Verify Mounts**: Check your disk partitions and file systems to ensure they are properly mounted and configured.

    ```bash
    df -hT
    cat /etc/fstab
    ```

* **Check YUM Repositories**: Navigate to the repository directory to confirm your repository files are in place.

    ```bash
    cd /etc/yum.repos.d/
    ```

---

### **2. Subscription and Package Management**

A valid Red Hat subscription is required for the upgrade.

1.  **Register and Auto-Attach Subscription**: **Replace the username and password with your own credentials**.

    ```bash
    subscription-manager register --username <YOUR_USERNAME> --password <YOUR_PASSWORD> --auto-attach
    ```

2.  **Verify Repositories**: Ensure your subscription is active and you have access to the Red Hat repositories.

    ```bash
    yum repolist
    ```

3.  **Set the Target Release**: Tell the subscription manager to use the RHEL 7.9 repositories, which is the last supported version for a direct upgrade to RHEL 8.

    ```bash
    subscription-manager release --set 7.9
    ```

4.  **Update All Packages**: It is a best practice to have all packages on your RHEL 7 system updated to their latest versions before beginning the upgrade.

    ```bash
    yum update
    reboot
    ```

---

### **3. Leapp Installation and Pre-Upgrade Analysis**

Install the Leapp utility and perform the initial pre-upgrade check to identify any potential issues.

1.  **Enable the Required Repository**: The Leapp package is located in the `rhel-7-server-extras-rpms` repository.

    ```bash
    subscription-manager repos --enable rhel-7-server-extras-rpms
    ```

2.  **Install Leapp**:

    ```bash
    yum install leapp
    ```

3.  **Perform First Pre-Upgrade Check**: Run the Leapp pre-upgrade command to analyze your system. This will generate a report listing potential issues that need to be fixed.

    ```bash
    leapp preupgrade --target 8.6
    ```

4.  **Resolve Issues from the Report**: Address any blocking issues found in the Leapp report (`/var/log/leapp/leapp-report.txt`). Common issues include unsupported kernel modules (`floppy`, `pata_acpi`) which can be removed.

    ```bash
    rmmod floppy
    rmmod pata_acpi
    ```

5.  **Confirm the Answerfile**: Leapp sometimes requires confirmation for certain changes. Edit the `answerfile` to confirm these changes.

    ```bash
    vim /var/log/leapp/answerfile
    ```
    Change the `confirm` line to `True`.

    ```
    confirm = True
    ```

6.  **Run Pre-Upgrade Check Again**: After resolving all issues, run the pre-upgrade command one more time to ensure all checks pass.

    ```bash
    leapp preupgrade --target 8.6
    ```
    If the report indicates everything is okay, you are ready to proceed with the upgrade.

---

### **4. Final Upgrade and Reboot**

1.  **Start the Upgrade Process**: Run the `leapp upgrade` command. The process will take some time as it downloads and installs all RHEL 8 packages.

    ```bash
    leapp upgrade --target 8.6
    ```

2.  **Reboot the System**: After the Leapp upgrade process finishes, reboot your system. The system will automatically boot into a temporary `initramfs` and complete the upgrade.

    ```bash
    reboot
    ```

3.  **Final Verification**: After the reboot, your system should be running RHEL 8. You can verify the new release with `cat /etc/redhat-release` and `uname -a`.
