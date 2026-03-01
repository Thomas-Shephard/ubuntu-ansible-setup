# Role: Security

This role handles the foundational system hardening of the Ubuntu server. It is designed to be run as the first step in provisioning a new server.

## Features
- **User Creation:** Creates a new, non-root administrative user (`security_new_user`) with `sudo` privileges and configures SSH key access.
- **SSH Hardening:** Changes the default SSH port, disables root login, disables password authentication, and prevents empty passwords.
- **Firewall (UFW):** Sets a default deny-all incoming policy and selectively opens necessary ports.
- **Intrusion Prevention:** Installs and configures `fail2ban` to protect the SSH daemon.
- **Automatic Updates:** Configures `unattended-upgrades` for automated security patching.

## Variables
These should be defined in `group_vars/all.yml` and `group_vars/secrets.yml`.

| Variable | Required | Description |
| :--- | :--- | :--- |
| `security_new_user` | Yes | The username for the new administrative user. |
| `security_new_user_password` | Yes | The plaintext password for the new user (will be hashed automatically). |
| `security_new_user_ssh_key` | Yes | The public SSH key for the new user. |
| `security_ssh_port` | Yes | The custom SSH port to listen on. |
