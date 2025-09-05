# Mule Standalone 3.9 Installation on RHEL 9

This document provides a step-by-step guide for installing Mule Standalone 3.9 on a Red Hat Enterprise Linux 9 system, including best practices for a production environment.

---

### **1. Hardware and Software Requirements**

Before starting the installation, ensure your system meets the following requirements:

#### **Hardware Requirements**
* **CPU**: Minimum 2 cores (4 cores recommended)
* **RAM**: Minimum 4 GB (8 GB recommended)
* **Disk Space**: Minimum 10 GB of free disk space

#### **Software Requirements**
* **Operating System**: RHEL 9
* **Java**: Mule 3.9 requires **Java 8 (JDK)**.

---

### **2. Install Java Development Kit (JDK)**

1.  **Update the system** to ensure all packages are up to date.

    ```bash
    sudo dnf update -y
    ```

2.  **Install OpenJDK 8**.

    ```bash
    sudo dnf install java-1.8.0-openjdk-devel -y
    ```

3.  **Verify the Java installation** by checking the version.

    ```bash
    java -version
    ```

---

### **3. Download and Install Mule Runtime**

1.  **Download the Mule Standalone 3.9.0** tarball.

    ```bash
    wget [https://repository.mulesoft.org/nexus/service/local/repositories/releases/content/org/mule/distributions/mule-standalone/3.9.0-20210217/mule-standalone-3.9.0-20210217.tar.gz](https://repository.mulesoft.org/nexus/service/local/repositories/releases/content/org/mule/distributions/mule-standalone/3.9.0-20210217/mule-standalone-3.9.0-20210217.tar.gz)
    ```

2.  **Extract the downloaded file**.

    ```bash
    tar -xvzf mule-standalone-3.9.0-20210217.tar.gz
    ```

3.  **Move the extracted directory** to a standard location like `/opt/`.

    ```bash
    sudo mv mule-standalone-3.9.0-20210217 /opt/mule
    ```

4.  **Set up environment variables** for easy access. Add `MULE_HOME` and update your `PATH` in your shell's profile.

    ```bash
    echo 'export MULE_HOME=/opt/mule' >> ~/.bashrc
    echo 'export PATH=$PATH:$MULE_HOME/bin' >> ~/.bashrc
    source ~/.bashrc
    ```

---

### **4. Start Mule Runtime**

1.  **Navigate to the Mule bin directory**.

    ```bash
    cd /opt/mule/bin
    ```

2.  **Start Mule**.

    ```bash
    ./mule
    ```

3.  **Verify that Mule is running** by checking the logs.

    ```bash
    tail -f /opt/mule/logs/mule.log
    ```

---

### **5. Best Practices**

This section outlines best practices for managing your Mule installation, including deployment and configuration.

#### **5.1. Deploying Applications**

* To deploy an application, simply place the `.jar` or `.zip` file into the `apps` directory within your Mule installation (`/opt/mule/apps`). Mule will automatically detect and deploy it.

#### **5.2. Setting Up Mule as a Systemd Service**

For production environments, it is a best practice to run Mule as a service so it starts automatically on boot and can be managed easily with standard system commands.

1.  **Create a `systemd` service file** for Mule.

    ```bash
    sudo nano /etc/systemd/system/mule.service
    ```

2.  **Add the following content** to the file. Remember to replace `ficonnect` with the user you want to run the service as.

    ```ini
    [Unit]
    Description=Mule Runtime
    After=network.target

    [Service]
    Type=simple
    User=ficonnect
    ExecStart=/opt/mule/bin/mule
    Restart=on-failure

    [Install]
    WantedBy=multi-user.target
    ```

3.  **Reload `systemd`** to recognize the new service.

    ```bash
    sudo systemctl daemon-reload
    ```

4.  **Start and Enable the service** to run on system boot.

    ```bash
    sudo systemctl start mule
    sudo systemctl enable mule
    ```

5.  **Check the service status** to confirm it is running correctly.

    ```bash
    sudo systemctl status mule
    ```

---

### **Conclusion**

You have successfully installed Mule Standalone 3.9 on RHEL 9 and configured it for a production environment. You can now begin deploying applications to your new runtime.
