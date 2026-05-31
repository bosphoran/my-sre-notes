---
layout: post
title: "Nginx & Apache2 Configuration"
date: 2026-03-13
categories: [linux]
---
##

Apache is your PHP application server.

- It’s the one that actually interprets PHP scripts, queries MySQL, and generates HTML responses. 
- Each Apache instance (on 8081 and 8082) serves a different site folder (mysite and mysite2).

NGINX in this setup is acting purely as a reverse proxy / load balancer. It doesn’t compile PHP or serve files directly. 

- Instead, when you visit <http://mysite.local>
- NGINX accepts the request on port 80.
- It decides which backend (Apache on 8081 or Apache on 8082) should handle the request — using round robin, least connections, or IP hash.
- It forwards the request to that backend.
- Apache generates the response (e.g., “Hello from backend 1”).
- NGINX passes that response back to your browser.

So the flow is:

> Browser → NGINX (port 80) → Apache backend (8081 or 8082) → PHP execution → Response → NGINX → Browser

✅ That’s why when you go directly to <mysite.local:8081>, you bypass NGINX and always hit backend #1.
✅ But when you go to <mysite.local> (no port), NGINX sits in front and distributes traffic between both backends.

---

🔄 Request Flow Pipeline

### Browser Request

- You type <http://mysite.local> into your browser.
- The browser resolves <mysite.local> to your Ubuntu server’s IP (thanks to your hosts file entry).

### NGINX Front‑End

- NGINX is listening on port 80.
- It receives the request first.
- NGINX looks at its upstream mysite_backend block and decides which backend to use (8081 or 8082).
- By default, it uses round robin: first request → backend 1, second request → backend 2, third request → backend 1, and so on.

### Proxy Forwarding

- NGINX forwards the request to the chosen backend:
- 127.0.0.1:8081 → Apache serving /var/www/html/mysite
- 127.0.0.1:8082 → Apache serving /var/www/html/mysite2

### Apache Backend

- Apache receives the forwarded request.
- It interprets any PHP scripts in that folder (mysite or mysite2).
- It generates the HTML response (e.g., “Hello from backend 1”).

### Response Back Through NGINX

- Apache sends the response back to NGINX.
- NGINX passes the response back to the browser.
- To the browser, it looks like the response came from mysite.local — it doesn’t know which backend was used.

### Browser Display

- The browser renders the page.
- If you refresh, NGINX may pick the other backend, so you see “Hello from backend 2” instead.

🧩 Key Roles

- Apache → PHP engine + web server for each backend folder.
- NGINX → Traffic director, deciding which backend gets each request.
- Browser → Only ever talks to NGINX, never directly to Apache (unless you specify :8081 or :8082).

> ✅ Apache compiles and serves PHP, while NGINX balances traffic between multiple Apache backends.

---

### 🚀 Step‑by‑Step Setup: Apache + PHP + NGINX Load Balancer

#### Install PHP and Apache

```bash
sudo apt update
sudo apt install apache2 php libapache2-mod-php -y
```

Verify Apache is running:

```bash
systemctl status apache2
```

#### Install MySQL (optional, if your site uses DB)

```bash
sudo apt install mysql-server -y
```

Install PHP‑MySQL extension:

```bash
sudo apt install php-mysql -y
sudo systemctl restart apache2
```

#### Install NGINX

```bash
sudo apt install nginx -y
```

Stop Apache from using port 80 (to free it for NGINX):

```bash
sudo nano /etc/apache2/ports.conf
```

Change:

```bash
Listen 80
```

to:

```bash
Listen 8081
Listen 8082
```

#### Configure Apache Virtual Hosts

Create site configs:

***mysite.conf***

```xml
<VirtualHost *:8081>
    ServerName mysite.local
    DocumentRoot /var/www/html/mysite

    <Directory /var/www/html/mysite>
        AllowOverride All
        Require all granted
    </Directory>
</VirtualHost>
```

---

***mysite2.conf***

```xml
<VirtualHost *:8082>
    ServerName mysite2.local
    DocumentRoot /var/www/html/mysite2
    <Directory /var/www/html/mysite2>
        AllowOverride All
        Require all granted
    </Directory>
</VirtualHost>
```

***Enable sites***

```bash
sudo a2ensite mysite.conf
sudo a2ensite mysite2.conf
sudo systemctl reload apache2
```

#### Configure NGINX Load Balancer

Create config:

```bash
sudo nano /etc/nginx/sites-available/mysite_lb.conf
```

Add:

```c
upstream mysite_backend {
    server 127.0.0.1:8081;
    server 127.0.0.1:8082;
}

server {
    listen 80;
    server_name mysite.local;

    location / {
        proxy_pass http://mysite_backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

        # Optional: show backend in response headers
        add_header X-Backend $upstream_addr;
    }
}
```

Enable config:

```bash
sudo ln -s /etc/nginx/sites-available/mysite_lb.conf /etc/nginx/sites-enabled/
sudo rm /etc/nginx/sites-enabled/default
sudo nginx -t
sudo systemctl reload nginx
```

#### Update Hosts File (on client PC)

On Windows:

```bash
C:\Windows\System32\drivers\etc\hosts
```

Add: (replace with your Ubuntu server IP).

```bash
192.168.1.50   mysite.local
```

#### Test

- Visit <http://mysite.local> → NGINX load balances between backend 1 and backend 2.
- Visit <http://mysite.local:8081> → always backend 1.
- Visit <http://mysite.local:8082> → always backend 2.

#### Monitor

Check logs:

```bash
sudo tail -f /var/log/nginx/access.log
sudo tail -f /var/log/apache2/access.log
```
