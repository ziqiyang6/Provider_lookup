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
   ssh zyang@<SERVER_IP>
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
2. Reload systemd and start the service:

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
   server {
       listen 80;
       server_name your.domain.com;

       location /static/ {
           alias /var/www/Provider_Lookup_Project/static/;
       }

       location /media/ {
           alias /var/www/Provider_Lookup_Project/media/;
       }

       location / {
           proxy_pass http://127.0.0.1:8000;
           proxy_set_header Host $host;
           proxy_set_header X-Real-IP $remote_addr;
           proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
           proxy_set_header X-Forwarded-Proto $scheme;
       }
   }
   ```
2. Enable and test Nginx config:

   ```bash
   sudo ln -s /etc/nginx/sites-available/provider-lookup \
                /etc/nginx/sites-enabled/
   sudo nginx -t
   sudo systemctl reload nginx
   ```
3. (Optional) Enable HTTPS with Let's Encrypt:

   ```bash
   sudo apt install certbot python3-certbot-nginx
   sudo certbot --nginx -d your.domain.com
   ```

---

## 4. Final Checks

* Visit `http://your.domain.com` to verify the application is live.
* Ensure static files load correctly.
* Check logs for any errors.

---

Feel free to customize paths, usernames, domain names, and service names as needed.
