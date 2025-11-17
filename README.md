# Ubuntu Setup with Ansible

This project provides a set of Ansible playbooks to automate the setup and configuration of a secure Ubuntu server, ready to host web applications in Docker containers.

It automates the following:
- **Initial Server Hardening:** Creates a new user, configures SSH, sets up a firewall, and installs security tools like Fail2ban.
- **Core Services:** Installs Docker, Docker Compose, and Nginx.
- **VPN Access:** Sets up a WireGuard VPN server to provide secure access to internal services.
- **Automated Deployments:** Configures a webhook listener to automatically deploy applications from a Git repository when changes are pushed.

## Requirements

- **Ansible:** Must be installed on your local machine.
- **Ubuntu:** A fresh installation of Ubuntu 24.04 or 25.04.
- **Root Access:** You will need root access to the server for the initial playbook run.

## Project Structure

- **`playbook.yml`**: The main playbook for initial setup and application deployment.
- **`add_client.yml`**: The playbook for adding new VPN clients.
- **`initial_setup/`**: An Ansible role for the initial server configuration.
- **`app_deployment/`**: An Ansible role for deploying an application.
- **`wireguard_add_client/`**: An Ansible role for adding new WireGuard clients.

---

## 1. Initial Server Setup
This section outlines the essential steps to prepare your Ubuntu machine, including user creation, SSH hardening, and core service installations.

### Steps

1.  **Create an Inventory File:**
    Create a file named `inventory` in the root of this project with the IP address of your server:
    ```ini
    [servers]
    your_server_ip ansible_user=root
    ```

2.  **Configure Server Variables:**
    Open `initial_setup/vars/main.yml` and configure the server settings:
    - `new_user`: The username for your new, non-root user.
    - `new_user_password`: The password for this user.
    - `new_user_ssh_key`: Your public SSH key. This will be used to log in as the new user.
    - `ssh_port`: The port for SSH (defaults to `22`).

3.  **Run the Playbook:**
    Execute the main playbook to set up the server:
    ```bash
    ansible-playbook -i inventory playbook.yml
    ```
    After this playbook completes, you should log in to your server using the new user and SSH key.

---

## 2. Deploying an Application
This section details how to deploy your web application using Docker Compose and set up automated deployments via GitHub webhooks.

### Steps

1.  **Configure Application Variables:**
    Open `app_deployment/vars/main.yml` and configure your application's settings:
    - `app_user`: The dedicated user for the application. Each application should have its own user for security and isolation.
    - `app_user_password`: The password for the application user.
    - `app_repo`: The Git repository URL of your application (e.g., `git@github.com:user/repo.git`).
    - `domain_name`: The domain name that will point to your application.
    - `webhook_secret`: A secret string for securing the GitHub webhook.

2.  **Run the Playbook Again:**
    Execute the main playbook again. This time, it will clone your repository and deploy your application using the `docker-compose.yml` file from the repository.
    ```bash
    ansible-playbook -i inventory playbook.yml
    ```

3.  **Set up the GitHub Webhook:**
    To enable automatic deployments on `git push`:
    - In your GitHub repository, go to **Settings > Webhooks > Add webhook**.
    - **Payload URL:** `http://<your_domain_name>:5000/webhook`
    - **Content type:** `application/json`
    - **Secret:** The value of `webhook_secret` from `app_deployment/vars/main.yml`.
    - **Events:** Select "Just the push event".

---

## 3. Managing VPN Access
This section explains how to manage access to your server's internal services using the WireGuard VPN.

### Steps

1.  **Generate a Key Pair:**
    On your client machine (e.g., your laptop or phone), generate a new WireGuard key pair.
    - **Linux/macOS:** `wg genkey | tee privatekey | wg pubkey > publickey`
    - **Windows/Mobile:** Use the WireGuard app to create a new tunnel, which will generate a key pair.

2.  **Configure Client Variables:**
    Open `wireguard_add_client/vars/main.yml` and add the new client's details:
    - `client_name`: A unique name for the client (e.g., `mylaptop`).
    - `client_public_key`: The public key you generated in the previous step.

3.  **Run the `add_client` Playbook:**
    ```bash
    ansible-playbook -i inventory add_client.yml
    ```
    This will generate a `<client_name>.conf` file in the project root.

4.  **Configure Your Client:**
    - Open the generated `<client_name>.conf` file.
    - Replace `YOUR_CLIENT_PRIVATE_KEY` with the private key you generated in step 1.
    - Import this modified file into your WireGuard client and activate the connection.