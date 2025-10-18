# HNG13 DevOps Stage 0 — Static Site Deployment (Vultr)

A minimal, production-hosted static web page demonstrating basic DevOps skills: version control, server provisioning, web serving, and reproducible deployment.

- Author: Fuhad Yusuf (@phurhard)
- Stack: HTML/CSS (static) + NGINX on a Vultr VPS
- Live Status: Successfully deployed on Vultr
- Deployed: 18/10/2025

## Project Overview

This repository contains a single-page site (`index.html`) that welcomes visitors and shows a deployment timestamp. The goal of Stage 0 is to deploy a static page to a cloud server and make it publicly accessible.

## Repository Structure

```
.
├── index.html   # Static landing page
└── README.md    # Project documentation (this file)
```

## Quick Start (Local)

Option 1 — Open directly:
- Double-click `index.html` to open it in your browser.

Option 2 — Serve locally (recommended):
```bash
# From the repo root:
python3 -m http.server 8080
# Then visit http://localhost:8080
```

## Deployment Guide (Vultr + NGINX)

The steps below assume a fresh Ubuntu server on Vultr (22.04/24.04). Adapt as needed for your environment.

1) SSH into the server
```bash
ssh <user>@<SERVER_IP>
# If using root (not recommended long-term):
# ssh root@<SERVER_IP>
```

2) Update and install NGINX
```bash
sudo apt update
sudo apt install -y nginx
```

3) Create a web root and upload the site
```bash
# On the server:
sudo mkdir -p /var/www/hng13-stage0
sudo chown -R www-data:www-data /var/www/hng13-stage0

# From your local machine, upload index.html:
scp index.html <user>@<SERVER_IP>:/tmp/index.html

# Back on the server:
sudo mv /tmp/index.html /var/www/hng13-stage0/index.html
sudo chown www-data:www-data /var/www/hng13-stage0/index.html
```

4) Configure an NGINX server block
Create a site config at `/etc/nginx/sites-available/hng13-stage0`:
```nginx
server {
    listen 80;
    server_name _;  # replace with your domain if you have one

    root /var/www/hng13-stage0;
    index index.html;

    access_log /var/log/nginx/hng13-access.log;
    error_log  /var/log/nginx/hng13-error.log;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

Enable the site, test, and reload:
```bash
sudo ln -sf /etc/nginx/sites-available/hng13-stage0 /etc/nginx/sites-enabled/hng13-stage0
sudo nginx -t
sudo systemctl reload nginx
```

5) (If UFW is enabled) Allow HTTP/HTTPS
```bash
sudo ufw allow 'Nginx Full'
sudo ufw status
```

6) Optional — HTTPS with Let’s Encrypt (Certbot)
```bash
sudo apt install -y certbot python3-certbot-nginx
# Replace with your domain(s):
sudo certbot --nginx -d example.com -d www.example.com
# Verify auto-renewal:
sudo systemctl status certbot.timer
```

7) Verify
- Visit http://<SERVER_IP> or your domain.
- Logs (if needed): `/var/log/nginx/hng13-access.log`, `/var/log/nginx/hng13-error.log`.

## Continuous Deployment (Optional)

Use GitHub Actions to deploy on push to `main`. You’ll need repository secrets:
- `VPS_HOST`, `VPS_USER`, `VPS_SSH_KEY` (private key with access to the server)

Example workflow: `.github/workflows/deploy.yml`
```yaml
name: Deploy to VPS

on:
  push:
    branches: [ main ]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Copy files via SCP
        uses: appleboy/scp-action@v0.1.7
        with:
          host: ${{ secrets.VPS_HOST }}
          username: ${{ secrets.VPS_USER }}
          key: ${{ secrets.VPS_SSH_KEY }}
          source: "index.html"
          target: "/var/www/hng13-stage0"

      - name: Reload NGINX
        uses: appleboy/ssh-action@v1.2.0
        with:
          host: ${{ secrets.VPS_HOST }}
          username: ${{ secrets.VPS_USER }}
          key: ${{ secrets.VPS_SSH_KEY }}
          script: sudo nginx -t && sudo systemctl reload nginx
```

## Maintenance Notes

- To update the visible deployment date, edit the `<p class="timestamp">` element in `index.html`.
- Keep your server updated: `sudo apt update && sudo apt upgrade -y`.
- Consider creating a non-root deploy user with limited privileges and SSH key authentication.
- If you later add a domain, update `server_name` and configure DNS A records.

## Troubleshooting

- 404 Not Found: Ensure `root` points to `/var/www/hng13-stage0` and `index index.html;` is set. Check `sudo nginx -t`.
- 403 Forbidden: Verify permissions/ownership:
  ```bash
  sudo chown -R www-data:www-data /var/www/hng13-stage0
  sudo find /var/www/hng13-stage0 -type d -exec chmod 755 {} \;
  sudo find /var/www/hng13-stage0 -type f -exec chmod 644 {} \;
  ```
- Port blocked: Confirm firewall rules and that your provider allows inbound 80/443.
- Still not working: Inspect `/var/log/nginx/hng13-error.log`.

## Credits

- HNG Internship 13 — DevOps Stage 0 challenge
- Hosted on Vultr
- Author: Fuhad Yusuf (GitHub: https://github.com/phurhard)
