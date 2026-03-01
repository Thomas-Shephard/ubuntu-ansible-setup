# Role: VPN

This role installs and configures a WireGuard VPN server to provide secure administrative access to the server's internal network (`10.0.0.0/24`).

## Features
- Installs WireGuard.
- Automatically generates secure X25519 private and public keys.
- Configures the `wg0` interface to listen on a custom UDP port.
- Enables IP forwarding and configures `iptables` NAT rules to allow traffic routing.

## Variables
These should be defined in `group_vars/all.yml`.

| Variable | Required | Description |
| :--- | :--- | :--- |
| `vpn_listen_port` | Yes | The UDP port that the WireGuard server listens on (e.g., 51820). |
