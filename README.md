# Ubuntu Setup with Ansible

This project provides a set of Ansible playbooks to automate the setup and configuration of a secure Ubuntu server, ready to host web applications in Docker containers.

It automates the following:
- **Initial Server Hardening:** Creates a new user, configures SSH, sets up a firewall, and installs security tools like Fail2ban.
- **Core Services:** Installs Docker (in rootless mode for enhanced security), Docker Compose, and Nginx.
- **VPN Access:** Sets up a WireGuard VPN server to provide secure access to internal services.
- **Automated Deployments:** Configures a centralized webhook listener to automatically deploy applications from a Git repository when changes are pushed.

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
    - `webhook_secret`: A secret string for securing the GitHub webhook. It is recommended to generate a long, random string for this value (e.g., using `openssl rand -hex 32` or a password manager).

3.  **Run the Playbook:**
    Execute the main playbook to set up the server:
    ```bash
    ansible-playbook -i inventory playbook.yml
    ```
    After this playbook completes, you should log in to your server using the new user and SSH key.

4.  **Update Inventory for Subsequent Runs:**
    After the initial setup, root login is disabled, and SSH password authentication is turned off. For all subsequent Ansible runs, you must update your `inventory` file to connect as the `new_user` with your SSH key.

    Modify your `inventory` file like this:
    ```ini
    [servers]
    your_server_ip ansible_user=your_new_user ansible_ssh_private_key_file=/path/to/your/private_key -e "ansible_port=your_ssh_port"
    ```
    Replace `your_new_user` with the value of `new_user`, `/path/to/your/private_key` with the path to your SSH private key, and `your_ssh_port` with the value of `ssh_port` if you changed it from the default 22.

---

## 2. Deploying an Application
This section details how to deploy your web application using Docker Compose and set up automated deployments via GitHub webhooks.

For each application you deploy, a dedicated, non-login system user will be created based on the repository name. This provides strong security and isolation between your applications.

### Steps

1.  **Configure Application Variables:**
    Open `app_deployment/vars/main.yml` and configure your application's settings:
    - `app_repo`: The Git repository URL of your application (e.g., `https://github.com/user/my-cool-app.git`). The user `my-cool-app` will be created automatically.
    - `app_domain_name`: The domain name that will point to your application.
    - `app_ports`: A list of environment variable names that your `docker-compose.yml` expects for port mappings. For each name in this list, the playbook will find a free host port and provide it as an environment variable. **`APP_PORT` is mandatory** as it is used by the Nginx reverse proxy.

      Example:
      ```yaml
      app_ports:
        - APP_PORT
        - PHPMYADMIN_PORT
      ```

2.  **Prepare Your `docker-compose.yml`:**
    Your application's `docker-compose.yml` must be structured to use the environment variables defined in `app_ports`.

    - **Web Service:** Use the `${APP_PORT}` environment variable for the host port mapping.
    - **Other Services (for VPN access):** Use their corresponding environment variables (e.g., `${PHPMYADMIN_PORT}`).
    - **Internal Services:** Services that do not need to be accessed from outside the Docker network should not have a `ports` section.

    Here is an example `docker-compose.yml` with a web server, a database, and phpMyAdmin:
    ```yaml
    version: '3.8'
    services:
      web:
        image: my-web-image
        restart: always
        ports:
          - "${APP_PORT:-80}:80"
        environment:
          # The web app connects to the database using the service name 'db'
          - DATABASE_HOST=db

      db:
        image: postgres:13
        restart: always
        volumes:
          - db_data:/var/lib/postgresql/data
        environment:
          - POSTGRES_PASSWORD=mysecretpassword

      phpmyadmin:
        image: phpmyadmin/phpmyadmin:latest
        restart: always
        ports:
          - "${PHPMYADMIN_PORT:-8080}:80"
        environment:
          - PMA_HOST=db
          - PMA_PORT=5432
          - PMA_USER=postgres
          - PMA_PASSWORD=mysecretpassword

    volumes:
      db_data:
    ```
    *(Note: The `:-8080` part provides a default host port for local development if the environment variable is not set.)*

3.  **Point A Records:**
    Before running the playbook, ensure that the A record for `{{ app_domain_name }}` (and `www.{{ app_domain_name }}` if applicable) in your DNS settings points to the IP address of your server.

4.  **Run the Playbook Again:**
    Execute the main playbook again. This time, it will create the application user, clone the repository, and deploy your application.
    ```bash
    ansible-playbook -i inventory playbook.yml
    ```

5.  **Set up the GitHub Webhook:**
    To enable automatic deployments on `git push`:
    - In your GitHub repository, go to **Settings > Webhooks > Add webhook**.
    - **Payload URL:** `http://<your_server_ip>:5000/webhook`
    - **Content type:** `application/json`
    - **Secret:** The value of `webhook_secret` from `initial_setup/vars/main.yml`.
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

---

## 4. Service Status Webpage
This playbook automatically creates a simple webpage that lists all deployed services and provides clickable links to access them.

This page is accessible **via VPN only** at `http://<your_server_ip>:5001`.
