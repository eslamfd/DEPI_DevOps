# Assignment 01

## Part 1: PHP & Nginx

### Step 1: Install Required Packages
Only install what you actually need:

```bash
sudo apt update
sudo apt install -y nginx php8.1-fpm
````

Screenshots: `1_php` , `1_nginx` , `1_php-f`
---

### Step 2: Configure PHP-FPM to Use a Socket

Check the pool configuration:

```bash
sudo nano /etc/php/8.1/fpm/pool.d/www.conf
```

Make sure these lines exist (uncomment or edit if necessary):

```
listen = /run/php/php8.1-fpm.sock
listen.owner = www-data
listen.group = www-data
listen.mode = 0660
```

Restart PHP-FPM to create the socket:

```bash
sudo service php8.1-fpm restart
```

Verify the socket exists:

```bash
ls -l /run/php/php8.1-fpm.sock
```

---

### Step 3: Configure Nginx to Use PHP-FPM

Edit the default Nginx site:

```bash
sudo nano /etc/nginx/sites-available/default
```

Minimal working config:

```nginx
server {
    listen 80 default_server;
    listen [::]:80 default_server;

    root /var/www/html;
    index index.php index.html index.htm;
    server_name _;

    location / {
        try_files $uri $uri/ =404;
    }

    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/run/php/php8.1-fpm.sock;
    }

    location ~ /\.ht {
        deny all;
    }
}
```

Test configuration:

```bash
sudo nginx -t
```

Start Nginx:

```bash
sudo systemctl start nginx
# or
sudo /usr/sbin/nginx
```

---

### Step 4: Test PHP

Create a minimal test site:

1. Create a folder for the site:

```bash
sudo mkdir -p /var/www/testsite
sudo chown -R $USER:$USER /var/www/testsite
```

2. Add a simple PHP page:

```bash
nano /var/www/testsite/index.php
```

```php
<?php
echo "<h1>Welcome to My Nginx + PHP Test Page!</h1>";
echo "<p>Current server time: " . date('Y-m-d H:i:s') . "</p>";
?>
```
**Check file:** `index.php`

3. Create an Nginx server block:

```bash
sudo nano /etc/nginx/conf.d/testsite.conf
```

```nginx
server {
    listen 80;
    server_name localhost;

    root /var/www/testsite;
    index index.php index.html;

    location / {
        try_files $uri $uri/ =404;
    }

    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/run/php/php8.1-fpm.sock;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
    }
}
```

**Check file:** `testsite.conf`

4. Test and restart Nginx:

```bash
sudo nginx -t
sudo systemctl restart nginx
```

5. Open in browser:
   Visit `http://localhost/index.php` (from WSL or Windows browser).

Screenshots : `1_local` , `1_my_test_page` 
---

### Summary : PHP-FPM & Nginx Cheat Sheet (Ubuntu 22.04)

**Socket Mode (default & recommended)**

```text
File: /etc/php/8.1/fpm/pool.d/www.conf
listen = /run/php/php8.1-fpm.sock
```

Check permissions:

```bash
ls -l /run/php/ | grep fpm
```

**Port Mode (TCP, useful for WSL/Docker setups)**

```text
listen = 127.0.0.1:9000
```

Restart PHP-FPM after editing:

```bash
sudo systemctl restart php8.1-fpm
```

**Nginx Config Examples**

* Socket version:

```nginx
fastcgi_pass unix:/run/php/php8.1-fpm.sock;
```

* Port version:

```nginx
fastcgi_pass 127.0.0.1:9000;
```

**Common Fixes for 502 Bad Gateway**

* Socket mode: ensure `/run/php/php8.1-fpm.sock` exists and is writable by `www-data`.
* Port mode: ensure PHP-FPM is listening (`ss -ltnp | grep 9000`).

Restart services if needed:

```bash
sudo systemctl restart php8.1-fpm nginx
```

---

## Part 2: Spring PetClinic

### Clone the Repository

```bash
git clone https://github.com/spring-projects/spring-petclinic
cd spring-petclinic
```

### 1. Run it on WSL
#### Step 1: Install Requirements

```bash
sudo apt update
sudo apt install -y openjdk-17-jdk maven
```

Check versions:

```bash
java -version
mvn -version
```

---

#### Step 2: Build and Run the App

Build the project:

```bash
./mvnw clean package
```

Run the app:

```bash
./mvnw spring-boot:run
# or
java -jar target/*.jar
```

Open in browser: `http://localhost:8080`

---

### 2. Containerization

**1. Normal Dockerfile**

* Image size: \~1.48 GB
* Tag: `original`

**2. Multistage Dockerfile**

* Image size: \~380 MB
* Tag: `latest`
* Pros: smaller final image, build caching, clean runtime stage

Screenshot : `2_images`
---

**3. Docker Compose + DB Config**

Files: `application-mysql.yml`, `application-postgres.yml`

**Why use `application-*.yml`?**

* Spring Boot picks up profile-specific configs (`application-{profile}.yml`) when a profile is active.
* Keeps configs organized and reusable.
* Each DB (Postgres/MySQL) has its own datasource settings.

**Activate profile example**:

```bash
SPRING_PROFILES_ACTIVE=postgres
```

* **MySql → on port 8081**
* **Postgres → on port 8080**

* Postgres → uses `application-postgres.yml`
* MySQL → uses `application-mysql.yml`
---

Screenshots: `2_containers` , `2_8080` , `2_8081`
---