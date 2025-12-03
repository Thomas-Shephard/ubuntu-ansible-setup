# Ubuntu Ansible Setup

This project provides a set of Ansible playbooks to automate the setup and configuration of a secure Ubuntu server, ready to host web applications in Docker containers.

## Requirements

- **Ansible:** Must be installed on your local machine.
  **Note:** Ansible does not support Windows as a control node. If you are on Windows, it is recommended to use Windows Subsystem for Linux (WSL) to run Ansible.
- **Ubuntu:** A fresh installation of Ubuntu 24.04 or later.
- **Initial Access:** You will need either direct root access or an account with `sudo` privileges on the server for the initial setup run.

---

## Usage

This project uses a two-playbook workflow: `setup.yml` for initial server configuration, and `deploy.yml` for deploying applications.

### Step 1: Initial Server Setup

This playbook performs the initial, one-time server setup. It hardens the server, installs necessary services (Nginx, WireGuard, etc.), and creates a new administrative user with `sudo` privileges.

1.  **Create an Inventory File:**
    Create a file named `inventory` in the root of this project with the IP address of your server. **You must connect as `root` or a user with `sudo` privileges for this first step.**
    ```ini
    [servers]
    your_server_ip ansible_user=your_initial_user
    ```

2.  **Configure Server Variables:**
    Open `initial_setup/vars/main.yml` and configure the server settings:
    - `new_user`: The username for your new, non-root administrative user.
    - `new_user_password`: The password for this user.
    - `new_user_ssh_key`: Your public SSH key. This will be the **only** way to log in as the new user.
    - `ssh_port`: The port for SSH (defaults to `22`).
    - `webhook_secret`: A secret string for securing the GitHub webhook.
    - `webhook_port`: The port for the webhook listener (defaults to `5000`).
    - `vpn_port`: The UDP port for WireGuard (defaults to `51820`).
    - `status_port`: The port for the service status webpage (defaults to `5001`).

3.  **Run the Setup Playbook:**
    Execute the `setup.yml` playbook. You will be prompted for the root user's password.
    ```bash
    ansible-playbook -i inventory setup.yml --ask-pass --ask-become-pass
    ```
    Initial setup is complete. The new user `new_user` has been created with sudo privileges.

4.  **Update Inventory for New User:**
    After the initial setup, password authentication for the new user is disabled. You must update your `inventory` file to connect as the new administrative user. Ensure you specify your SSH private key file and the correct SSH port.

    ```ini
    [servers]
    your_server_ip ansible_user=new_user ansible_ssh_private_key_file=/path/to/your/private_key ansible_port=your_ssh_port
    ```

### Step 2: Deploying an Application

After the initial setup is complete, you can deploy one or more applications by running the `deploy.yml` playbook.

1. **Configure Application Variables:**
    Open `app_deployment/vars/main.yml` and configure your application's settings:
    - `app_repo`: The Git repository URL of your application (e.g., `https://github.com/user/my-cool-app.git`).
    - `app_domain_name`: The domain name that will point to your application.
    - `app_ports`: A list of environment variable names that your `docker-compose.yml` expects for port mappings. **`APP_PORT` is mandatory** as it is used by the Nginx reverse proxy.

2. **Point DNS Records:**
    Ensure that the A record for your application's domain in your DNS settings points to the IP address of your server.

3. **Run the Deployment Playbook:**
    Execute the `deploy.yml` playbook.
    ```bash
    ansible-playbook -i inventory deploy.yml
    ```
    If your application's repository contains a `.env.example` file, Ansible will prompt you to enter values for each variable.

4. **Set up the GitHub Webhook:**
    To enable automatic deployments on `git push`:
    - In your GitHub repository, go to **Settings > Webhooks > Add webhook**.
    - **Payload URL:** `http://<your_server_ip>:5000/webhook`
    - **Content type:** `application/json`
    - **Secret:** The value of `webhook_secret` from `initial_setup/vars/main.yml`.
    - **Events:** Select "Just the push event".

---

### Other Playbooks

#### Managing VPN Access (`add_client.yml`)

This playbook adds a new client to the WireGuard VPN.

1.  **Configure Client Variables:**
    Open `wireguard_add_client/vars/main.yml` and set:
    - `client_name`: A unique name for your device.
    - `client_public_key`: The public key of your WireGuard client.

2.  **Run the Playbook:**
    ```bash
    ansible-playbook -i inventory add_client.yml
    ```
    This will generate a `<client_name>.conf` file in the project root.

3.  **Finalize & Import:**
    Open the generated `<client_name>.conf` file, replace `YOUR_CLIENT_PRIVATE_KEY` with your client's private key, and then import the configuration into your WireGuard client application.

#### Removing an Application (`remove_app.yml`)

This playbook removes a deployed application, including the user, files, and Nginx configuration.

-   You must provide the application's repository URL as an extra variable.
-   Run this playbook as your administrative user (`new_user`).

```bash
ansible-playbook -i inventory remove_app.yml -e "app_repo=https://github.com/user/my-cool-app.git"
```

---
### Technical Details

#### Application Isolation

For each application you deploy, a dedicated, non-login system user will be created based on the repository name. The application is cloned into this `new_user`'s home directory, and rootless Docker is used to run the application's containers. This provides strong security and isolation between your applications.

#### Secret Management

If your application repository contains a `.env.example` file, the `deploy.yml` playbook will automatically detect it and interactively prompt you for each variable. This creates a `.env` file in the application directory, which is used by Docker Compose. This ensures your sensitive application secrets are not hardcoded.

**Note:** If new keys are added to `.env.example`, a webhook-triggered deployment will fail. You must manually run `ansible-playbook -i inventory deploy.yml` to be prompted for the new secret values.

#### Service Status Webpage

A simple status page is created that lists all deployed services and provides clickable links to access them. This page is accessible **via VPN only** at `http://<your_server_ip>:{{ status_port }}`.