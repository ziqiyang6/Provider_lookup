# Deployment Guide for Django Project

This README provides a step-by-step template to deploy your Django project from GitHub to a production server using **systemd** and **nginx**.

---

## Prerequisites

* A Linux server with SSH access (e.g., Ubuntu 20.04+).
* A non-root user with sudo privileges (e.g., `zyang`).
* Python 3.12+, virtualenv, and pip installed.
* Git installed.
* nginx installed.
* systemd (built into most modern Linux distributions).

---

## 1. Clone from GitHub

1. SSH into your server:

   ```bash
   ssh username@<SERVER_IP>
   ```
2. Navigate to the directory where you want to deploy:

   ```bash
   cd /var/www/
   ```
3. Clone your repository:

   ```bash
   git clone https://github.com/ziqiyang6/Provider_Lookup_Project.git
   cd Provider_Lookup_Project
   ```
4. Create and activate a virtual environment:

   ```bash
   python3 -m venv .venv
   source .venv/bin/activate
   ```
5. Install dependencies:

   ```bash
   pip install -r requirements.txt
   ```

---

## 2. Configure systemd Service

1. Create a service file at `/etc/systemd/system/provider-lookup.service`, template:

   ```ini
   [Unit]
   Description=Provider Lookup Django Service
   After=network.target

   [Service]
   User=zyang
   Group=www-data
   WorkingDirectory=/var/www/Provider_Lookup_Project
   EnvironmentFile=/var/www/Provider_Lookup_Project/.env
   ExecStart=/var/www/Provider_Lookup_Project/.venv/bin/gunicorn \
       provider_system.wsgi:application \
       --bind 127.0.0.1:8000 \
       --workers 3
   Restart=on-failure
   KillMode=process
   LimitNOFILE=65535
   SyslogIdentifier=provider-lookup

   [Install]
   WantedBy=multi-user.target
   ```
2. Reload systemd and start the service if you have permission, else ask Mr.Counts or Aaron:

   ```bash
   sudo systemctl daemon-reload
   sudo systemctl start provider-lookup
   sudo systemctl enable provider-lookup
   ```
3. Check status and logs:

   ```bash
   sudo systemctl status provider-lookup
   sudo journalctl -u provider-lookup -f
   ```

---

## 3. Configure Nginx

1. Create an Nginx config at `/etc/nginx/sites-available/provider-lookup`:

   ```nginx
   # Virtual Host configuration for Shamash demo project
#

upstream django {
  server 127.0.0.1:8000 max_fails=5 fail_timeout=60s;
}

server {
  server_name pms-demo.okc-labs.us;

  location / {
    allow all;

    # Proxy Headers
    proxy_http_version 1.1;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header Host $http_host;
    proxy_set_header X-Cluster-Client-Ip $remote_addr;

    # The Important Websocket Bits!
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";

    proxy_pass http://django;
  }

    listen 443 ssl; # managed by Certbot
    ssl_certificate /etc/letsencrypt/live/shamash.binarypaean.com/fullchain.pem; # managed by Certbot
    ssl_certificate_key /etc/letsencrypt/live/shamash.binarypaean.com/privkey.pem; # managed by Certbot
    include /etc/letsencrypt/options-ssl-nginx.conf; # managed by Certbot
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem; # managed by Certbot

}


server {
    if ($host = pms-demo.okc-labs.us) {
        return 301 https://$host$request_uri;
    } # managed by Certbot


  server_name pms-demo.okc-labs.us;
  listen 80;
    return 404; # managed by Certbot


}
   ```
2. Enable and test Nginx config:

   ```bash
   sudo ln -s /etc/nginx/sites-available/provider-lookup \
                /etc/nginx/sites-enabled/
   sudo nginx -t
   sudo systemctl reload nginx
   ```
