# Ubuntu Ansible Setup

This project provides a set of Ansible playbooks to automate the setup and configuration of a secure Ubuntu server, ready to host web applications in Docker containers.

## Requirements

- **Ansible:** Must be installed on your local machine.
  **Note:** Ansible does not support Windows as a control node. If you are on Windows, it is recommended to use Windows Subsystem for Linux (WSL) to run Ansible.
- **Ansible Collections:** You must install the required collections for this project.
  ```bash
  ansible-galaxy collection install -r requirements.yml
  ```
- **Ubuntu:** A fresh installation of Ubuntu 24.04 or later.
- **Initial Access:** You will need either direct root access or an account with `sudo` privileges on the server for the initial setup run.
- **Domain Name:** A registered domain name is required for Nginx and Let's Encrypt SSL certificates.

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

    **Sensitive Variables:**
    Copy the example secrets file to a real secrets file (which is ignored by Git), and encrypt it using Ansible Vault. This ensures your credentials are never committed in plaintext.

    ```bash
    cp group_vars/secrets.example.yml group_vars/secrets.yml
    ansible-vault encrypt group_vars/secrets.yml
    ```
    *You will be prompted to create a vault password. Keep this safe.*

    Open `group_vars/secrets.yml` (using `ansible-vault edit group_vars/secrets.yml`) and set:
    - `new_user_password`: The password for this user.
    - `webhook_secret`: A secret string for securing the GitHub webhook.
    - `github_pat`: (Optional) A GitHub Personal Access Token.
    - `new_user_ssh_key`: Your public SSH key.

    **General Variables:**
    Open `group_vars/all.yml` and configure:
    - `new_user`: The username for your new, non-root administrative user.
    - `ssh_port`: The port for SSH (defaults to `22`).
    - `vpn_port`: The UDP port for WireGuard (defaults to `51820`).

3.  **Run the Setup Playbook:**
    Execute the `setup.yml` playbook. Because you encrypted your secrets, you must include the `--ask-vault-pass` flag.
    ```bash
    ansible-playbook -i inventory setup.yml --ask-vault-pass --ask-pass --ask-become-pass
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
    - `app_branch`: The branch to deploy (defaults to `main`).
    - `app_domain_name`: The domain name that will point to your application.
    - `certbot_email`: Email address for Let's Encrypt expiration notifications.
    - `app_deployment_context`: (Optional) The context label for GitHub status updates (e.g., `deployment/production`). Defaults to `deployment/production`.
    - `app_exposed_services`: A list defining which services from your `docker-compose.yml` should be accessible from the internet. 
      ```yaml
      app_exposed_services:
        - name: "frontend"                            # Name of the service in docker-compose.yml
          port: "80"                                  # The port the app runs on INSIDE the container (e.g., 3000 for React, 80 for Nginx). We ignore/wipe host mappings (like 8080:80).
          rule: "Host(`{{ app_domain_name }}`)"       # Traefik routing rule
        - name: "api"
          port: "8080"
          rule: "Host(`api.{{ app_domain_name }}`)"   # E.g., Expose API on a subdomain
      ```

    > **Note:** You do **not** need to modify your `docker-compose.yml` for this deployment. Ansible automatically injects a `docker-compose.override.yml` that removes any hardcoded host port mappings and seamlessly bridges traffic from Traefik to your specified internal ports.

2. **Point DNS Records:**
    Ensure that the following DNS records point to your server's IP address:
    - **A Record:** `@` (or your subdomain) -> `your_server_ip`
    - **CNAME/A Record:** `webhook` -> `your_server_ip` (required for automatic deployments)
    > **Tip:** Verify your DNS has propagated using `dig` or `nslookup` before running the playbook. If propagation is incomplete, the SSL certificate request will fail.

3. **Run the Deployment Playbook:**
    Execute the `deploy.yml` playbook.
    ```bash
    ansible-playbook -i inventory deploy.yml --ask-become-pass
    ```
    If your application's repository contains a `.env.example` file, Ansible will prompt you to enter values for each variable.

4. **Set up the GitHub Webhook:**
    To enable automatic deployments on `git push`:
    - In your GitHub repository, go to **Settings > Webhooks > Add webhook**.
    - **Payload URL:** `https://webhook.<your_app_domain>/webhook`
    - **Content type:** `application/json`
    - **Secret:** The value of `webhook_secret` from `group_vars/secrets.yml`.
    - **SSL verification:** Select **"Enable SSL verification"** (Recommended).
    - **Events:** Select "Just the push event".

---

### Other Playbooks

#### Managing VPN Access (`add_client.yml`)

This playbook adds a new client to the **WireGuard VPN**. WireGuard runs on UDP port `51820` (by default) and is used to securely access internal services (like the Traefik Dashboard or internal app endpoints) without exposing them to the public internet.

1.  **Configure Client Variables:**
    Open `wireguard_add_client/vars/main.yml` and set:
    - `client_name`: A unique name for your device.
    - `client_public_key`: The public key of your WireGuard client.
      **Note:** WireGuard keys are based on Curve25519 and are not compatible with typical RSA or SSH keys.

2.  **Run the Playbook:**
    ```bash
    ansible-playbook -i inventory add_client.yml --ask-become-pass
    ```
    This will generate a `<client_name>.conf` file in the project root.

3.  **Finalize & Import:**
    Open the generated `<client_name>.conf` file, replace `YOUR_CLIENT_PRIVATE_KEY` with your client's private key, and then import the configuration into your WireGuard client application.
    > **Note:** The generated configuration uses **Split Tunneling**. Only traffic to the internal network (`10.0.0.0/24`) is routed through the VPN. Your normal internet traffic remains direct.

### Advanced Usage: Ansible Tags

The playbooks are heavily tagged, allowing you to run specific parts of the setup without executing the entire playbook. This is useful for updates or specific administrative tasks.

```bash
# Only run the VPN setup tasks (e.g., after updating wireguard config)
ansible-playbook -i inventory setup.yml --tags "vpn" --ask-vault-pass --ask-become-pass

# Only update security configurations (UFW, fail2ban)
ansible-playbook -i inventory setup.yml --tags "security" --ask-vault-pass --ask-become-pass

# Re-run just the Traefik proxy setup
ansible-playbook -i inventory setup.yml --tags "traefik" --ask-vault-pass --ask-become-pass
```

Available tags in `setup.yml`: `security`, `docker`, `traefik`, `vpn`, `dns`, `tools`.

---

### Troubleshooting & Operations

#### Viewing Application Logs
Each application runs as a systemd user service. To view the logs for a specific application:
```bash
# Replace <app_user> with the user created for your app (usually the repo name)
# Get the User ID (UID) first:
id -u <app_user>

# Then view logs using the UID:
journalctl _UID=<UID> _SYSTEMD_USER_UNIT=app.service -f
```

#### Backups & Recovery
Automatic backups of application configurations (`.env`, `docker-compose.yml`) and `backups/` directories run nightly.
- **Location:** `/var/backups/apps/`
- **Format:** `apps_backup_YYYY-MM-DD.tar.gz`
- **Retention:** Backups older than 7 days are automatically deleted.

**To Restore:**
1. Extract the tarball: `tar -xzf /var/backups/apps/apps_backup_<DATE>.tar.gz`
2. Copy the files back to the application directory: `/home/<app_user>/app/`

#### Manual Deployment / Updates Failed
If a webhook deployment fails (e.g., due to new variables in `.env.example` that require input), the system will reject the update to prevent crashing the app.
**Fix:** Run the Ansible deployment playbook manually to interactively provide the new values:
```bash
ansible-playbook -i inventory deploy.yml --ask-become-pass
```

#### Checking Webhook Service
If webhooks are not triggering updates:
1. Check the service status: `systemctl status webhook`
2. View webhook logs: `journalctl -u webhook -f`
3. Ensure the GitHub Webhook delivery history shows a `200` or `202` response.

---
### Technical Details

#### Application Isolation

For each application you deploy, a dedicated, non-login system user will be created based on the repository name. The application is cloned into this `new_user`'s home directory, and rootless Docker is used to run the application's containers. This provides strong security and isolation between your applications.

#### Secret Management

If your application repository contains a `.env.example` file, the `deploy.yml` playbook will automatically detect it and interactively prompt you for each variable. This creates a `.env` file in the application directory, which is used by Docker Compose. This ensures your sensitive application secrets are not hardcoded.

**Note:** If new keys are added to `.env.example`, a webhook-triggered deployment will fail. You must manually run `ansible-playbook -i inventory deploy.yml` to be prompted for the new secret values.

#### Traefik Proxy & Dashboard

This setup uses Traefik as the central reverse proxy. Traefik automatically discovers new applications via the File Provider and handles Let's Encrypt SSL certificates automatically.

**Accessing the Dashboard:**
A built-in Traefik Dashboard is available to monitor the status of all active routers and services. For security, it is not exposed to the public internet. 

To view the dashboard:
1. Connect to the WireGuard VPN.
2. Navigate to `http://traefik.internal` in your browser.
