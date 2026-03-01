# Role: App Deployment

This role automates the deployment of a containerized application from a Git repository onto the server. It handles everything from user isolation to reverse proxy routing.

## Features
- **Strict Isolation:** Creates a dedicated, non-login system user for each application to prevent lateral movement.
- **Rootless Docker:** Configures and runs a rootless Docker daemon specifically for the application user.
- **Git Deployment:** Clones the specified repository (supporting both public and private repos via Deploy Keys).
- **Environment Prompting:** Automatically detects `.env.example` files and interactively prompts the user for secrets, merging them with auto-generated passwords.
- **Traefik Auto-Discovery:** Dynamically injects a `docker-compose.override.yml` to remove host port mappings and bridges the application to the Traefik Unix socket network without modifying the developer's source code.

## Variables
These should be defined in `roles/app_deployment/vars/main.yml`.

| Variable | Required | Description |
| :--- | :--- | :--- |
| `app_deployment_repo` | Yes | The Git URL of the application. |
| `app_deployment_deploy_key` | No | The SSH private key if using a private repository. |
| `app_deployment_branch` | Yes | The branch to deploy. |
| `app_deployment_domain_name` | Yes | The primary domain for the application. |
| `app_deployment_certbot_email` | Yes | The email for Let's Encrypt notifications. |
| `app_deployment_exposed_services` | Yes | A list of services from the `docker-compose.yml` to expose to the internet via Traefik. |
