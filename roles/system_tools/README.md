# Role: System Tools

This role installs operational tools required for the continuous integration and maintenance of the server.

## Features
- **Webhook Listener:** Installs a Python-based webhook server that listens for GitHub `push` events and automatically triggers a `git pull` and Docker Compose rebuild for deployed applications.
- **Automated Backups:** Installs a nightly cron job that archives the configuration (`.env` and `docker-compose.yml`) of all deployed applications and retains them for 7 days.

## Variables
These should be defined in `group_vars/secrets.yml`.

| Variable | Required | Description |
| :--- | :--- | :--- |
| `webhook_secret` | Yes | A secret string used to validate the HMAC signature of incoming GitHub webhooks. |
| `github_pat` | No | A Personal Access Token used if the server needs authenticated access to GitHub APIs. |
