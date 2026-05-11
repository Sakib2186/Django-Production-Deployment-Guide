````md
# Deploy Multiple Django Applications on VPS using Nginx + Gunicorn

## Server Requirements

- Ubuntu 22.04+
- Root or sudo access
- Domain/subdomain pointed to your VPS
- MySQL/PostgreSQL database
- Python 3.10+
- Nginx

Optional:

- Webuzo / CyberPanel / cPanel (for DNS + database management)

---

# Example Structure

We will deploy multiple Django apps like:

```bash
/home/django/project-1
/home/django/project-2
````

Each project will have its own:

* Virtual environment
* Gunicorn service
* Socket
* Nginx config

---

# 1. Install Required Packages

Update system:

```bash
sudo apt update && sudo apt upgrade -y
```

Install Python + pip:

```bash
sudo apt install python3 python3-pip python3-venv -y
```

Optional symlink:

```bash
sudo ln -s /usr/bin/pip3 /usr/bin/pip
```

Install virtualenv:

```bash
python3 -m pip install virtualenv
```

Install nginx:

```bash
sudo apt install nginx -y
```

---

# 2. Upload Django Projects

Example:

```bash
/home/django/project-1/
/home/django/project-2/
```

---

# 3. Setup Virtual Environment

Move into projects directory:

```bash
cd /home/django
```

Create virtualenv:

```bash
python3 -m virtualenv project-1_venv
```

Activate:

```bash
source project-1_venv/bin/activate
```

Install dependencies:

```bash
pip install -r project-1/requirements.txt
```

Install gunicorn if missing:

```bash
pip install gunicorn
```

---

# 4. Configure Django

Move into project:

```bash
cd /home/django/project-1
```

Run migrations:

```bash
python manage.py makemigrations
python manage.py migrate
```

Collect static files:

```bash
python manage.py collectstatic --noinput
```

Create superuser:

```bash
python manage.py createsuperuser
```

---

## Django settings.py

Update:

```python
DEBUG = False

ALLOWED_HOSTS = [
    "your-domain.com",
    "www.your-domain.com"
]
```

---

# 5. Create Logs Directory

```bash
sudo mkdir -p /home/django/project-1/logs
sudo chmod -R 775 /home/django/project-1/logs
```

---

# 6. Create Systemd Socket

Create:

```bash
sudo nano /etc/systemd/system/project-1.socket
```

Content:

```ini
[Unit]
Description=project-1 gunicorn socket

[Socket]
ListenStream=/run/project-1.sock

[Install]
WantedBy=sockets.target
```

---

# 7. Create Systemd Service (WSGI)

Create:

```bash
sudo nano /etc/systemd/system/project-1.service
```

Content:

```ini

[Unit]
Description=gunicorn daemon
Requires=project-1.socket
After=network.target

[Service]
User=root
Group=root
WorkingDirectory=/home/django/project-1
Environment="DJANGO_SETTINGS_MODULE=project-1.setting"
ExecStart=/home/django/project-1/venv/bin/gunicorn \
          --access-logfile /home/django/project-1/logs/gunicorn-access.log \
          --error-logfile /home/django/project-1/logs/gunicorn-error.log \
          --log-level debug \
          --capture-output \
          --workers 3 \
          --timeout 300 \
          --bind unix:/run/hrm-api.tclinformatix.sock \
          tcl_hrm_api.wsgi:application
Restart=always
RestartSec=3

[Install]
WantedBy=multi-user.target
```

---

# 8. Start Service

```bash
sudo systemctl daemon-reload

sudo systemctl start project-1.socket
sudo systemctl enable project-1.socket

sudo systemctl start project-1.service
sudo systemctl enable project-1.service
```

Check status:

```bash
sudo systemctl status project-1.service
```

---

# 9. Configure Nginx

Create config:

```bash
sudo nano /etc/nginx/sites-available/project-1
```

Content:

```nginx
server {
    listen 80;

    server_name your-domain.com www.your-domain.com;

    location /static/ {
        alias /home/django/project-1/staticfiles/;
    }

    location /media/ {
        alias /home/django/project-1/media/;
    }

    location / {
        include proxy_params;
        proxy_pass http://unix:/run/project-1.sock;
    }
}
```

Enable:

```bash
sudo ln -s /etc/nginx/sites-available/project-1 /etc/nginx/sites-enabled/
```

Test:

```bash
sudo nginx -t
```

Restart:

```bash
sudo systemctl restart nginx
```

---

# 10. Enable SSL

Install certbot:

```bash
sudo apt install certbot python3-certbot-nginx -y
```

Generate SSL:

```bash
sudo certbot --nginx -d your-domain.com -d www.your-domain.com
```

---

# Deploy Additional Projects

Repeat same steps for:

* project-2
* project-3

Just change:

* project name
* socket name
* service name
* domain

Example:

```bash
project-2.sock
project-2.service
```

---

# ASGI Deployment (Django Channels / WebSockets)

Use this only if your project needs:

* WebSockets
* Real-time notifications
* Chat system
* Background live updates

---

## Install ASGI Packages

Activate environment:

```bash
source /home/django/project-1_venv/bin/activate
```

Install:

```bash
pip install channels channels-redis daphne uvicorn
```

---

## Update systemd Service

Edit:

```bash
sudo nano /etc/systemd/system/project-1.service
```

Replace `wsgi` with `asgi`:

```ini
[Unit]
Description=gunicorn daemon
Requires=project-1.socket
After=network.target

[Service]
User=root
Group=root
WorkingDirectory=/home/django/project-1
Environment="DJANGO_SETTINGS_MODULE=project-1.setting"
ExecStart=/home/django/project-1/venv/bin/gunicorn \
          --access-logfile /home/django/project-1/logs/gunicorn-access.log \
          --error-logfile /home/django/project-1/logs/gunicorn-error.log \
          --log-level debug \
          --capture-output \
          --workers 3 \
          -k uvicorn.workers.UvicornWorker \
          --timeout 300 \
          --bind unix:/run/hrm-api.tclinformatix.sock \
          tcl_hrm_api.asgi:application
Restart=always
RestartSec=3

[Install]
WantedBy=multi-user.target
```

---

## Update Nginx for WebSocket Support

Edit nginx config:

```bash
sudo nano /etc/nginx/sites-available/project-1
```

Add:

```nginx
location /ws/ {
    include proxy_params;
    proxy_pass http://unix:/run/project-1.sock;

    proxy_http_version 1.1;

    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
}
```

---

## Reload Services

```bash
sudo systemctl daemon-reload

sudo systemctl restart project-1.socket
sudo systemctl restart project-1.service

sudo nginx -t
sudo systemctl restart nginx
```

---

# Logs & Debugging

Nginx logs:

```bash
sudo tail -f /var/log/nginx/error.log
```

Systemd logs:

```bash
journalctl -u project-1.service -f
```
