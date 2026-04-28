# The Ultimate Ubuntu Deployment Manual for Modern Web Apps

Welcome! This guide is your go-to reference for deploying any modern web application on an Ubuntu server. Whether you're working with **Next.js**, **Frappe**, **Express**, or even a simple **static HTML site**, this manual will walk you through every step, from initial server setup to a secure, production-ready launch.

We'll cover all the core tools and concepts in detail, ensuring you have a deep understanding of how everything works together. Let's get started!

---

## 1. Server Setup: Your Digital Foundation

Your journey begins with a fresh Ubuntu server. A proper setup is the foundation for a stable and secure deployment.

### Choosing and Connecting to a VPS

First, you'll need a **Virtual Private Server (VPS)**, which is your own private slice of a powerful computer in the cloud. Popular choices include:

* **DigitalOcean:** Known for its simplicity and excellent documentation.
* **AWS (Amazon Web Services):** Offers a powerful suite of tools, including the EC2 service for virtual servers.
* **Linode:** A solid alternative with competitive pricing.

Once you've created your VPS, you'll receive an IP address. You connect to your server from your local machine using **SSH (Secure Shell)**, a secure protocol for remote access.

```bash
ssh root@YOUR_SERVER_IP
```

### Initial Server Setup: The Security Checklist

Running everything as the `root` user is like leaving the master key to your entire building under the doormat—it's incredibly risky. We'll create a new user with administrative privileges instead.

1.  **Create a New User:**
    ```bash
    adduser your_username
    ```
    Follow the prompts to set a strong password.

2.  **Grant Administrative Privileges:**
    This command adds your new user to the `sudo` group, allowing them to perform administrative tasks by prefixing commands with `sudo`.
    ```bash
    usermod -aG sudo your_username
    ```

3.  **Set Up a Basic Firewall (UFW):**
    A firewall is a digital security guard that controls incoming and outgoing traffic. UFW (Uncomplicated Firewall) makes this easy.
    ```bash
    # Allow SSH connections so you don't lock yourself out
    ufw allow OpenSSH
    
    # Allow all standard web traffic (HTTP and HTTPS)
    ufw allow 'Nginx Full'
    
    # Turn the firewall on
    ufw enable
    ```

4.  **Log in as Your New User:**
    From now on, always use your new, secure user to log in.
    ```bash
    ssh your_username@YOUR_SERVER_IP
    ```

### Folder Structure Best Practices

Where should you put your app's code? Sticking to conventions makes your server easier to manage.

* **/var/www/yourdomain.com**: This is the **industry standard and highly recommended** location for web-facing files. It keeps your application code separate from system and user files.
* `/srv/yourdomain.com`: An alternative for service-related data.
* `/home/your_username/apps`: Convenient for personal projects, but less standard for production.

For this guide, we'll use the `/var/www/` structure. Let's create it:
```bash
sudo mkdir -p /var/www/[yourdomain.com/html](https://yourdomain.com/html)
sudo chown -R $USER:$USER /var/www/[yourdomain.com/html](https://yourdomain.com/html)
```

---

## 2. Database Setup

Most web applications need a database. Here’s how to set up **PostgreSQL**, a powerful and popular choice.

### A. Install PostgreSQL
```bash
sudo apt update
sudo apt install postgresql postgresql-contrib
```

### B. Create a Database and User
1.  **Log into PostgreSQL:**
    ```bash
    sudo -u postgres psql
    ```
2.  **Create a database:** This is the container for your app's data.
    ```sql
    CREATE DATABASE myappdb;
    ```
3.  **Create a new user and password:** It's a security best practice to have a separate user for each application.
    ```sql
    CREATE USER myappuser WITH PASSWORD 'your_secure_password';
    ```
4.  **Grant privileges** to your new user on your new database:
    ```sql
    GRANT ALL PRIVILEGES ON DATABASE myappdb TO myappuser;
    ```
5.  **Exit psql:**
    ```sql
    \q
    ```

### C. Connect Your App
Your application will connect to the database using a **connection string**. In your app's environment variables (e.g., a `.env` file), you would add:
```
DATABASE_URL="postgresql://myappuser:your_secure_password@localhost:5432/myappdb"
```

---

## 3. Nginx Web Server: The Traffic Controller

Nginx is the heart of your deployment, acting as a high-performance web server and reverse proxy.

### What is a Reverse Proxy?

A reverse proxy is like a receptionist. It sits in front of your applications and directs incoming traffic to the correct one based on the domain or path requested. This is essential for hosting multiple sites or apps on a single server.

### Setting Up Nginx

1.  **Install Nginx:**
    ```bash
    sudo apt update
    sudo apt install nginx
    ```

2.  **Nginx Configuration Files:**
    Nginx configurations live in two directories:
    * `/etc/nginx/sites-available/`: This is your library of all possible website configurations.
    * `/etc/nginx/sites-enabled/`: This is where you activate a configuration by creating a symbolic link to a file in `sites-available`.

### Nginx for a Static HTML Site

1.  Create a configuration file:
    ```bash
    sudo nano /etc/nginx/sites-available/yourdomain.com
    ```

2.  Paste this configuration:
    ```nginx
    server {
        listen 80;
        server_name yourdomain.com [www.yourdomain.com](https://www.yourdomain.com);
    
        root /var/www/[yourdomain.com/html](https://yourdomain.com/html);
        index index.html index.htm;
    
        location / {
            try_files $uri $uri/ =404;
        }
    }
    ```

3.  **Activate it:**
    ```bash
    sudo ln -s /etc/nginx/sites-available/yourdomain.com /etc/nginx/sites-enabled/
    ```

### Nginx for a Node.js App (Next.js, Express)

Your Node.js app runs on a specific port (e.g., 3000). Nginx will proxy requests to it.
```nginx
upstream my_node_app {
    server 127.0.0.1:3000;
}

server {
    listen 80;
    server_name yourdomain.com;

    location / {
        proxy_pass http://my_node_app;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

### Nginx for a Python App (Frappe, Flask)

Similarly, your Python app runs on a port (e.g., 8000).
```nginx
upstream my_python_app {
    server 127.0.0.1:8000;
}

server {
    listen 80;
    server_name yourdomain.com;

    location / {
        proxy_pass http://my_python_app;
        proxy_set_header Host $host;
        # ... other headers
    }
}
```

### Hosting Multiple Sites on the Same Server

Create a separate configuration file for each domain in `/etc/nginx/sites-available/` and enable each one:
```bash
sudo ln -s /etc/nginx/sites-available/yourdomain.com /etc/nginx/sites-enabled/
sudo ln -s /etc/nginx/sites-available/anotherdomain.com /etc/nginx/sites-enabled/
```

### Hosting Multiple Apps on the Same Domain

Use `location` blocks to route paths to different apps:
```nginx
server {
    listen 80;
    server_name yourdomain.com;

    # Route /app to Frappe
    location /app {
        proxy_pass [http://127.0.0.1:8000](http://127.0.0.1:8000);
        # ... headers
    }

    # Route everything else to Next.js
    location / {
        proxy_pass [http://127.0.0.1:3000](http://127.0.0.1:3000);
        # ... headers
    }
}
```

---

## 4. SSL Setup with Let's Encrypt

HTTPS is non-negotiable. Certbot makes it easy to get free SSL certificates.

1.  **Install Certbot:**
    ```bash
    sudo snap install --classic certbot
    sudo ln -s /snap/bin/certbot /usr/bin/certbot
    ```

2.  **Obtain and Install a Certificate:**
    Certbot will automatically detect your domains from your Nginx files and configure them.
    ```bash
    sudo certbot --nginx
    ```
    Follow the prompts, and be sure to choose the option to **redirect** HTTP to HTTPS.

3.  **Auto-Renewal:**
    Certbot automatically sets up a cron job or systemd timer to renew your certificates. You can test the renewal process with:
    ```bash
    sudo certbot renew --dry-run
    ```

---

## 5. PM2 for Node.js Apps

PM2 is a production-grade process manager for Node.js.

1.  **Install PM2:**
    ```bash
    sudo npm install -g pm2
    ```

2.  **Start Your App:**
    ```bash
    # For a Next.js app
    pm2 start npm --name "my-next-app" -- start
    
    # For an Express app
    pm2 start app.js --name "my-express-api"
    ```

3.  **Auto-Start on Reboot:**
    This command generates a startup script for your system.
    ```bash
    pm2 startup
    ```
    It will give you a command to run, which you should execute. Then, save your current process list:
    ```bash
    pm2 save
    ```

4.  **PM2 Logs and Monitoring:**
    * **View logs:** `pm2 logs`
    * **Monitor CPU/memory:** `pm2 monit`
    * **Stop an app:** `pm2 stop my-next-app`
    * **Restart an app:** `pm2 restart my-next-app`

---

## 6. Frappe App Deployment

Here are some specifics for Frappe.

* **Running on Port 8000:** By default, `bench start` runs on port 8000. In production, Supervisor manages this.
* **Hosting Multiple Frappe Sites:** Your Nginx configuration should use the `X-Frappe-Site-Name` header to tell Frappe which site to serve:
    ```nginx
    proxy_set_header X-Frappe-Site-Name $host;
    ```
* **Routing `/app` to Frappe:** As shown in the Nginx section, a `location /app` block is all you need.
* **Log Files:** Frappe logs are located in `~/frappe-bench/logs/`.

---

## 7. Next.js / React Deployment

* **`npm run build` and `npm start`:** Always use these commands for production. `npm run build` creates an optimized production build, and `npm start` (which runs `next start`) serves it.
* **Using PM2:** The command `pm2 start npm --name "my-app" -- start` is the standard way to run a Next.js app in production.
* **Serving on Port 3000:** Next.js defaults to port 3000, which is perfect for proxying from Nginx.

---

## 8. Static Site Deployment

* **Where to Place Files:** `/var/www/yourdomain.com/html` is the standard.
* **Nginx Caching:** To improve performance, add a `location` block to your Nginx config to cache static assets:
    ```nginx
    location ~* \.(jpg|jpeg|png|gif|ico|css|js)$ {
        expires 7d;
    }
    ```

---

## 9. General Tips and Troubleshooting

### Common Port Issues

* **Port 80/443 in use:** This usually means another web server (like Apache) is running. Stop and disable it (`sudo systemctl stop apache2`).
* **App port in use:** If port 3000 is taken, you can run your Next.js app on a different port: `npm start -- -p 3001`.

### Permissions and Ownership

If you get permission errors, ensure your user owns the application files:
```bash
sudo chown -R your_username:your_username /path/to/your/app
```

### Basic Troubleshooting

* **`nginx -t`:** Your best friend. Always run this before reloading Nginx to check for syntax errors.
* **`systemctl status nginx`:** Shows if Nginx is running and any recent errors.
* **`pm2 logs`:** Shows the output of your Node.js app, including any startup errors.
* **Nginx Log Files:**
    * **Access Log:** `/var/log/nginx/access.log`
    * **Error Log:** `/var/log/nginx/error.log` (Check this first for 502/403/404 errors).

### Debugging Nginx Errors

* **502 Bad Gateway:** Nginx can't reach your app. Is your app running? Is it on the correct port?
* **403 Forbidden:** Nginx doesn't have permission to read your files. Check file ownership and permissions (`chmod`).
* **404 Not Found:** The file doesn't exist at the path Nginx is looking. Check your `root` directive.

---

## 10. Bonus

### Deploying an Express.js API

The process is nearly identical to Next.js.

1.  Run your Express app on a port (e.g., 5000).
2.  Start it with PM2: `pm2 start server.js --name "my-api"`
3.  Set up an Nginx reverse proxy to port 5000. For APIs, it's common to use a subdomain (`api.yourdomain.com`) or a path (`yourdomain.com/api`).

### Connecting a Full-Stack App

In a typical full-stack setup, your frontend (Next.js) will make API calls to your backend (Express/Frappe). Your Nginx reverse proxy is what makes this seamless. You can configure a `location /api` block to route all API calls to your backend server, while the `location /` block serves your frontend.

This guide covers all the essential steps to confidently deploy your web applications. Happy deploying!

---

## Author Details

* **Name:** Kartikey Mishra
* **Guide Created:** August 4, 2025
