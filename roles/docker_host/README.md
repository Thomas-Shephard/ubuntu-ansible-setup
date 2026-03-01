# Role: Docker Host

This role is responsible for installing the core Docker daemon on the server.

## Features
- Installs prerequisite packages (apt-transport-https, ca-certificates, etc.).
- Securely downloads and configures the official Docker GPG key.
- Configures the official Docker APT repository.
- Installs the `docker-compose-plugin`.

## Notes
This role installs the *system-level* Docker components. It does **not** configure the daemon to run applications. The `ubuntu-ansible-setup` architecture relies on Rootless Docker instances isolated per user, which is handled dynamically by the `app_deployment` role using the `rootless_docker.yml` tasks included within this directory.
