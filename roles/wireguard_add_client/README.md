# Role: WireGuard Add Client

This utility role is used to provision access for a new peer on the existing WireGuard VPN server.

## Features
- Validates the formatting of the provided client name and public key.
- Parses the existing `/etc/wireguard/wg0.conf` to automatically determine the next available `10.0.0.X` IP address.
- Safely appends the new peer configuration to the server without disrupting existing connections.
- Generates a complete `<vpn_client_name>.conf` file locally that can be imported directly into a WireGuard client app (configured for split tunneling).

## Variables
These should be defined in `roles/wireguard_add_client/vars/main.yml`.

| Variable | Required | Description |
| :--- | :--- | :--- |
| `vpn_client_name` | Yes | A unique identifier for the client (alphanumeric). |
| `vpn_client_public_key` | Yes | The 44-character Base64 encoded Curve25519 public key. |
