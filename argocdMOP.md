# Argo CD Setup and Management MOP

This document outlines the steps for setting up the Argo CD CLI, managing user accounts, and following best practices for a secure environment.

---

### **1. Argo CD CLI Installation**

This section covers how to install the `argocd` command-line tool on your local machine.

1.  **Download the latest CLI binary** using `curl`.

    ```bash
    curl -sSL -o argocd-linux-amd64 [https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64](https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64)
    ```

2.  **Install the binary** to a directory in your system's PATH. This makes the `argocd` command available from any terminal location.

    ```bash
    sudo install -m 555 argocd-linux-amd64 /usr/local/bin/argocd
    ```

3.  **Verify the installation** by checking the client version.

    ```bash
    argocd version --client
    ```

---

### **2. User Management MOP**

This procedure details how to change the admin password and create new users with the `argocd` CLI.

#### **2.1. Change Admin Password**

1.  **Log in to the Argo CD server** using the initial password you retrieved from the `argocd-initial-admin-secret` Kubernetes secret.

    ```bash
    argocd login <ARGOCD_SERVER_IP_OR_HOSTNAME>
    ```

2.  **Update the admin password** by running the following command and following the on-screen prompts.

    ```bash
    argocd account update-password
    ```

#### **2.2. Create a New User Account**

Argo CD manages users through the **`argocd-cm`** and **`argocd-rbac-cm`** ConfigMaps.

1.  **Generate a password hash** for the new user. You can get a hash by changing the admin password and then inspecting the `argocd-secret` for the new hash. For example, the `password` field will contain the new hash.

2.  **Edit the `argocd-cm` ConfigMap** to add the new user with the generated password hash.

    ```bash
    kubectl edit configmap argocd-cm -n argocd
    ```

    Add the following to the `data` section, replacing `newusername` with your desired username and `your_password_hash` with the hash you generated.

    ```yaml
    data:
      users.newusername: |
        password: your_password_hash
    ```

3.  **Define user permissions** by editing the `argocd-rbac-cm` ConfigMap.

    ```bash
    kubectl edit configmap argocd-rbac-cm -n argocd
    ```

    Add a policy to the `data.policy.csv` section, giving your new user the appropriate role (e.g., `role:admin`, `role:readonly`, etc.).

    ```yaml
    data:
      policy.csv: |
        p, newusername, role:admin, *, *, *
    ```

---

### **3. Best Practices for Argo CD**

This section provides a summary of best practices for a secure and scalable Argo CD setup.

* **Integrate with OIDC**: Instead of creating local accounts, use an existing identity provider (like Okta, Google, or GitHub) with OIDC for centralized authentication and user management. This is the most secure and scalable method.

* **Git is the Source of Truth**: All application changes should be made in your Git repository. Argo CD should automatically detect and synchronize these changes, making your Git repository the single source of truth for your cluster's state.

* **Use Sync Waves**: For applications with dependencies (e.g., a database before a web application), use sync waves to ensure resources are created in the correct order, preventing deployment failures.

* **Configure Notifications**: Set up notifications to alert your team via Slack or email about important events, such as application sync failures or health degradations.
