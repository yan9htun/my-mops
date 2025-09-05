# Method of Procedure: Creating a RHEL 8.10 Kickstart ISO

This document outlines the steps to create a customized RHEL 8.10 Kickstart ISO.

## 1. Prerequisites: Install Required Tools

Before you begin, ensure you have the necessary tools installed. On a Debian-based system (like Ubuntu), you can install them with the following command:

```bash
sudo apt-get install -y genisoimage xorriso p7zip-full
```

## 2. Create the Kickstart Configuration File (`ks.cfg`)

Create a file named `ks.cfg` with your desired kickstart configuration. Here is an example configuration:

```
#version=RHEL8
# Use graphical install
graphical
# System language
lang en_US.UTF-8
# Keyboard layouts
keyboard --vckeymap=us --xlayouts='us'
# System timezone
timezone Asia/Yangon --isUtc
# Root password
rootpw --iscrypted $6$XvB/eFtfn.OpobDG$jMDuQkFGYv1jK/KmpjTtT1klLh9AbrjRnGPdcw9dkIfWeO2WLII4.3q5ypJw/MWevrhmVojeg9P6vN2oO1tKc1
# User creation
user --name=systeam --groups=wheel --password=$6$XvB/eFtfn.OpobDG$jMDuQkFGYv1jK/KmpjTtT1klLh9AbrjRnGPdcw9dkIfWeO2WLII4.3q5ypJw/MWevrhmVojeg9P6vN2oO1tKc1 --iscrypted
# System services
services --enabled="sshd,cockpit" --disabled="firewalld"
# SELinux configuration
selinux --disabled
# Firewall configuration
firewall --disabled
# Network information
network  --bootproto=dhcp --device=ens192 --onboot=on --ipv6=auto --activate
network  --hostname=rhel8-template
# Reboot after installation
reboot
# System bootloader configuration
bootloader --location=mbr --boot-drive=sda
# Clear the Master Boot Record
zerombr
# Partition clearing information
clearpart --all --initlabel
# Disk partitioning information
part /boot --fstype="xfs" --ondisk=sda --size=1024
part pv.01 --fstype="lvmpv" --ondisk=sda --size=1 --grow
volgroup vg_root --pesize=4096 pv.01
logvol /  --fstype="xfs" --name=lv_root --vgname=vg_root --size=1 --grow
logvol swap  --fstype="swap" --name=lv_swap --vgname=vg_root --size=4096

%packages
@^server-with-gui
@performance
@system-tools
@network-tools
@security-tools
chrony
openssh-server
vim
wget
curl
sudo
cockpit
subscription-manager
%end

%post --log=/root/ks-post.log
echo "Starting post-installation script"

# Configure chrony
echo "server 172.24.11.31 iburst" >> /etc/chrony.conf
echo "server 172.24.11.32 iburst" >> /etc/chrony.conf

# Enable and start chronyd
systemctl enable --now chronyd
systemctl restart chronyd

# Log chrony status
chronyc sources > /root/chronyc-sources.log
chronyc tracking >> /root/chronyc-sources.log

echo "Finished post-installation script"
%end
```

## 3. Prepare the Build Environment

1.  **Create a working directory**:

    ```bash
    mkdir kickstart_build
    ```

2.  **Extract the RHEL 8.10 ISO**:

    ```bash
    7z x rhel-8.10-x86_64-dvd.iso -o./kickstart_build/
    ```

3.  **Copy the Kickstart File**:

    ```bash
    cp ks.cfg kickstart_build/
    ```

## 4. Modify Boot Configuration

1.  **Modify `isolinux.cfg` for BIOS booting**:

    Open `kickstart_build/isolinux/isolinux.cfg` and add a new label for the kickstart installation, setting it as the default.

2.  **Modify `grub.cfg` for UEFI booting**:

    Open `kickstart_build/EFI/BOOT/grub.cfg` and add a new menu entry for the kickstart installation, setting it as the default.

## 5. Create the Kickstart ISO

Run the following command to create the new bootable ISO:

```bash
genisoimage -o rhel-8.10-kickstart.iso -b isolinux/isolinux.bin -c isolinux/boot.cat --no-emul-boot --boot-load-size 4 --boot-info-table -J -R -v -T -joliet-long -V 'RHEL-8-10-0-BaseOS-x86_64' kickstart_build/
```

## 6. Best Practices

*   **Version Control:** Store your `ks.cfg` file in a version control system like Git to track changes.
*   **Test in a VM:** Always test your kickstart file and the resulting ISO in a virtual machine before using it on physical hardware.
*   **Minimal Packages:** In the `%packages` section, start with a minimal set of packages and add only what is necessary to reduce the image size and attack surface.
*   **Security:** Avoid storing plaintext passwords in your kickstart file. Use encrypted passwords as shown in the example.
*   **Post-install Scripts:** Use the `%post` section for configuration tasks that are difficult to do with standard kickstart commands. Keep these scripts as simple as possible and log their output for debugging.
