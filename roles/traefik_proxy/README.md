# Role: Traefik Proxy

This role provisions the central Traefik edge router that handles all HTTP/HTTPS traffic for the deployed applications.

## Features
- Runs a dedicated, rootless Traefik container isolated under a `traefik_proxy` system user.
- Configures the **File Provider** to dynamically watch `/etc/traefik/dynamic` for new application routing rules.
- Sets up automatic Let's Encrypt SSL certificate generation using the ACME HTTP challenge.
- Secures the Traefik Dashboard, making it available only via the internal WireGuard network at `traefik.internal`.

## Architecture
The proxy communicates with downstream applications exclusively via Unix Sockets located in `/opt/traefik/sockets/`. This completely eliminates host port collisions and allows applications to be fully isolated.
