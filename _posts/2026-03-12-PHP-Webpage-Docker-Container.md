---
layout: post
title: "PHP Webpage in Docker Container"
date: 2026-03-12
categories: [docker]
---

## 🐳 Step‑by‑Step: Create a PHP Webpage Container

### 1. Project Setup

```bash
mkdir -p ~/docker-projects/php-demo
cd ~/docker-projects/php-demo
```

Create your PHP file:

```bash
nano index.php
```

Example content:

```php
<?php
    echo "Hello from Dockerized PHP!";
?>
```

### 2. Dockerfile

Save this as php-demo.dockerfile inside ~/docker-projects/php-demo/:

```bash
FROM php:8.2-apache # Use official PHP image with Apache
WORKDIR /var/www/html # Set working directory inside the container
COPY index.php /var/www/html/ # Copy your PHP site into the container
EXPOSE 80 # Expose port 80 inside the container
```

### 3. Build the Image

Run from inside the project folder:

```bash
docker build -t php-demo -f php-demo.dockerfile .
```

### 4. Run the Container

Map container port 80 → host port 8080:

```bash
docker run -d -p 8080:80 --name php-demo-container php-demo
```
	
Test in browser:

```html
http://<server-ip>:8080	<!-- http://192.168.77.50:8080
```

---

## 🔄 Updating Your PHP Page

### 1. Edit Your PHP File

Make changes in ~/docker-projects/php-demo/index.php.
Example:

```php
<?php
    echo "Updated PHP page!";
?>
```

### 2. Rebuild the Image

```bash
cd ~/docker-projects/php-demo
docker build -t php-demo -f php-demo.dockerfile .
```

### 3. Restart the Container

Remove the old one:

```bash
docker rm -f php-demo-container
```

Run the new one:

```bash
docker run -d -p 8080:80 --name php-demo-container php-demo
```

✅ Now your updated PHP page is served at:

```html
http://<server-ip>:8080
```

---

## 🐳 Step‑by‑Step: Run PHP Container with Mount

### 1. Stop and Remove Old Container

If you already have a container running:

```bash
docker rm -f php-demo-container
```

### 2. Run New Container with Mount

From inside your project folder (~/docker-projects/php-web-demo), run:

```bash
docker run -d -p 8080:80 -v ~/docker-projects/php-web-demo:/var/www/html --name php-web-demo-container php:8.2-apache
```

```bash
-p 8080:80 # maps host port 8080 to container port 80.
-v ~/docker-projects/php-web-demo:/var/www/html # mounts your actual project folder into the container’s Apache document root.
--name php-web-demo-container # gives the container a clear name.
```

### 🔄 Step 4: Update Workflow

- Edit ~/docker-projects/php-web-demo/index.php on your host.
- Refresh the browser.
- Changes appear instantly — no rebuild needed.
