# Role: DNS

This role configures an internal DNS resolver using `dnsmasq` that operates strictly over the WireGuard VPN interface.

## Features
- Installs `dnsmasq`.
- Configures it to listen exclusively on the `wg0` interface address (`10.0.0.1`).
- Defines an internal TLD (`.internal`) to resolve explicitly to the server IP.
- Allows DNS queries through UFW only from the VPN interface, ensuring the resolver cannot be queried publicly or used in a DNS amplification attack.

## Dependencies
This role depends on the `vpn` role being executed first, as it requires the `wg0` interface and its associated IP address to be available.
