---
layout: post
title: "PHP + MySQL Container Setup"
date: 2026-03-13
categories: [docker, mysql, php]
---

In this setup, we will create a docker container that runs a web application powered by PHP & MySQL.

## Create Project Folder

```bash
mkdir -p ~/docker-projects/php-mysql-demo
cd ~/docker-projects/php-mysql-demo
```

## Create index.php

```php
<?php
$servername = "192.168.77.50"; // Ubuntu host IP
$username   = "zafar";
$password   = "your_password";
$dbname     = "mysite_db";

$conn = new mysqli($servername, $username, $password, $dbname);

if ($conn->connect_error) {
    die("Connection failed: " . $conn->connect_error);
}

$sql    = "SELECT id, name, email FROM users";
$result = $conn->query($sql);

if ($result->num_rows > 0) {
    echo "<h1>Users Table</h1><ul>";
    while($row = $result->fetch_assoc()) {
        echo "<li>" . $row["id"]. " - " . $row["name"]. " (" . $row["email"]. ")</li>";
    }
    echo "</ul>";
} else {
    echo "No users found.";
}

$conn->close();
?>
```

## Create Dockerfile for PHP + MySQL Demo

```bash
FROM php:8.2-apache # Use official PHP image with Apache
RUN docker-php-ext-install mysqli # Install mysqli extension for MySQL connectivity
WORKDIR /var/www/html # Set working directory
COPY . /var/www/html/ # Copy project files into container
EXPOSE 80 # Expose Apache port
```

## Build the image

```bash
docker build -t php-mysql-demo .
```

## Run PHP Container with Mount

```bash
docker run -d -p 8083:80 -v ~/docker-projects/php-mysql-demo:/var/www/html --name php-mysql-demo-container php:8.2-apache	
```

## Install MySQL Extension in Container

```bash
docker exec -it php-mysql-demo-container bash
docker-php-ext-install mysqli
docker-php-ext-enable mysqli
exit
docker restart php-mysql-demo-container
```

---

## MySQL CLI Commands

## Log in as Root (Ubuntu default)

```bash	
sudo mysql
```

## Create Database

```sql
CREATE DATABASE mysite_db;
```

## Create Table

```sql
USE mysite_db;
CREATE TABLE users (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(100),
    email VARCHAR(100)
);
```

## Insert Sample Data

```sql
INSERT INTO users (name, email) VALUES
('Alice', 'alice@example.com'),
('Bob', 'bob@example.com');
```

## Create User

```sql
CREATE USER 'zafar'@'%' IDENTIFIED BY 'your_password';
```

## Grant Privileges

```sql
GRANT ALL PRIVILEGES ON mysite_db.* TO 'zafar'@'%';
FLUSH PRIVILEGES;
```

## Update User Password

```sql
ALTER USER 'zafar'@'%' IDENTIFIED BY 'new_password';
```

## Verify Users

```sql
SELECT user, host FROM mysql.user WHERE user='zafar';
```

## Test Connection

```sql
From host:
    mysql -h 192.168.77.50 -u zafar -p

From container:
    docker exec -it php-mysql-demo-container bash
    mysql -h 192.168.77.50 -u zafar -p
```

## 🔑 Summary

- Docker side: Run PHP container with project folder mounted, install mysqli.
- MySQL side: Create database, table, user, grant privileges, and allow remote connections (bind-address=0.0.0.0).
- Testing: Always verify with mysql -h 'host-ip' -u 'user' -p from both host and container.

- Build first, then run with mount.
- If you change only your PHP files, you don’t need to rebuild — just edit in your mounted folder and refresh the browser.
- If you change the Dockerfile (e.g., add extensions), you must rebuild the image and restart the container.
- If you want to re‑run after changes:
  
  - docker rm -f php-mysql-demo-container
  - docker run -d -p 8081:80 -v ~/docker-projects/php-mysql-demo:/var/www/html --name php-mysql-demo-container php-mysql-demo
