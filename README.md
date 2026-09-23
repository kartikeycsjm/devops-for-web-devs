# The Developer's Guide to Ubuntu Web Deployments

When you first start deploying websites to a VPS, the configuration can feel like a lot of magic. "Reverse Proxy", "Ports", "Daemons"... what does it all actually mean?

This guide breaks down exactly how to put your code on the internet from absolute scratch. Whether you are deploying a simple React app, a full-stack Next.js application, or a complex Python/Node backend, this guide explains not just *how* to do it, but *why* it works.

---

## 1. Server Setup: Your Digital Foundation

Your journey begins with a fresh Ubuntu server. Running everything as the `root` user is like leaving the master key to your entire building under the doormat—it's incredibly risky. 

Here is how to set up your server securely from day one.

### The Security Checklist
Connect to your server (`ssh root@YOUR_SERVER_IP`) and run these commands to secure it:

1. **Create a New User & Grant Admin Privileges:**
   ```bash
   adduser your_username
   usermod -aG sudo your_username
   ```
2. **Set Up a Basic Firewall (UFW):**
   A firewall is your digital security guard. We only want to let in SSH (so you can log in) and web traffic (HTTP/HTTPS).
   ```bash
   ufw allow OpenSSH
   ufw allow 'Nginx Full'
   ufw enable
   ```
3. **Log in as Your New User:**
   Exit the server, and from now on, always connect as your new user: `ssh your_username@YOUR_SERVER_IP`

### Folder Structure Best Practices
Where should you put your app's code? 
* **/var/www/yourdomain.com**: This is the industry standard location for web-facing files. It keeps your application code cleanly separated from system files.

Let's create it and take ownership of it:
```bash
sudo mkdir -p /var/www/yourdomain.com
sudo chown -R $USER:$USER /var/www/yourdomain.com
```

---

## 2. The Nginx Mental Model

Nginx is the heart of your deployment. It acts as the receptionist for your server. When a user types your domain name into their browser, Nginx answers the door on Port 80 and decides where to send the user based on your configuration.

Nginx generally has two completely different jobs depending on what framework you are deploying.

### Job A: Nginx as a "Web Server" (For Static Apps like React/Vite)
When you build a standard React app (`npm run build`), it just generates a folder of static files (`.html`, `.css`, `.js`). There is no active code running. Because there is no active background process, Nginx simply acts as a traditional file server and hands those files to the user.

### Job B: Nginx as a "Reverse Proxy" (For Node.js/Next.js/Python)
Frameworks like Next.js, Express, or Frappe generate data *on the fly*. Because of this, they require an active server process running continuously in the background on a private Port (like `3000` or `8000`). 
For these apps, Nginx acts as a **Reverse Proxy** (a middleman). It catches the web traffic on Port 80, quietly knocks on the door of Port 3000, gets the webpage, and hands it back to the user securely.

---

## 3. Deploying a Static App (React/Vite)

Because there is no running server process needed, deployment is incredibly simple.

1. Build the files: `npm run build`
2. Place the `dist` or `build` folder inside `/var/www/yourdomain.com/`
3. Create your Nginx config: `sudo nano /etc/nginx/sites-available/yourdomain.com`

**The Config:**
```nginx
server {
    listen 80;
    server_name yourdomain.com;
    
    # Point Nginx to the folder containing your built files
    root /var/www/yourdomain.com/dist;
    index index.html;

    # This handles client-side routing. If a user directly visits a link like /about, 
    # Nginx will fallback to index.html so React Router can handle it without a 404 error.
    location / {
        try_files $uri $uri/ /index.html;
    }
}
```

---

## 4. Deploying a Server-Side App (Next.js/Express)

To host a Node.js app, you'll use **PM2** (a process manager) to keep your app running in the background. If you just run `npm start` in your terminal and close the window, your website dies. PM2 prevents this.

1. Install PM2: `sudo npm install -g pm2`
2. Start your app: `pm2 start npm --name "my-app" -- start`
3. Ensure PM2 restarts if the server reboots:
   ```bash
   pm2 startup
   pm2 save
   ```

Now, configure Nginx to Reverse Proxy traffic to your running PM2 app:

**The Config:**
```nginx
server {
    listen 80;
    server_name yourdomain.com;

    # Forward all incoming traffic to the app running on port 3000
    location / {
        proxy_pass http://localhost:3000;
        
        # Standard boilerplate to ensure headers and WebSockets pass through correctly
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
    }
}
```

---

## 5. The "Hybrid" Architecture (Frontend & Backend on One Domain)

Often, you'll have a frontend (React/Next.js) and a custom backend API (Python/Node/Frappe). 

Instead of putting your backend on a separate subdomain like `api.yourdomain.com`, you can configure Nginx to host them **both on the exact same domain**. Nginx will simply look at the URL path and act like a traffic cop routing cars to different lanes.

**Why do this?**
1. **No CORS Errors:** Because the frontend and backend share the exact same domain, you avoid all Cross-Origin Resource Sharing (CORS) headaches permanently.
2. **Cleaner Code:** Your frontend can simply fetch `/api/users` instead of typing out full URLs.

**The Config:**
```nginx
server {
    listen 80;
    server_name yourdomain.com;

    # 1. THE BACKEND: Any URL starting with /api/ is sent to your backend (Port 8000)
    location ^~ /api/ {
        proxy_pass http://localhost:8000;
        proxy_set_header Host $host;
    }

    # 2. THE FRONTEND: Everything else is sent to your Next.js frontend (Port 3000)
    location / {
        proxy_pass http://localhost:3000;
        proxy_set_header Host $host;
    }
}
```

---

## 6. Securing it with SSL (Let's Encrypt)

HTTPS is non-negotiable for modern web apps. Certbot makes it incredibly easy to get free SSL certificates and will automatically update your Nginx files for you.

1. **Install Certbot:**
   ```bash
   sudo snap install --classic certbot
   sudo ln -s /snap/bin/certbot /usr/bin/certbot
   ```
2. **Obtain and Install a Certificate:**
   ```bash
   sudo certbot --nginx -d yourdomain.com
   ```
   Follow the prompts, and Certbot will automatically rewrite your Nginx config to listen on Port 443 (HTTPS) and redirect all HTTP traffic to HTTPS.

---

## 7. Database Setup (PostgreSQL)

If your app needs a database, PostgreSQL is a powerful, industry-standard choice. It's a security best practice to have a separate user and password for each application.

```bash
# 1. Install Postgres
sudo apt update && sudo apt install postgresql postgresql-contrib

# 2. Log into the Postgres prompt
sudo -u postgres psql
```

Inside the SQL prompt, run these commands:
```sql
CREATE DATABASE myappdb;
CREATE USER myappuser WITH PASSWORD 'your_secure_password';
GRANT ALL PRIVILEGES ON DATABASE myappdb TO myappuser;
\q
```
Your app can now connect using this string: `postgresql://myappuser:your_secure_password@localhost:5432/myappdb`

---

## 8. General Tips and Troubleshooting

Whenever you update an Nginx configuration file, **always** run this command before reloading:
```bash
sudo nginx -t
```
This tests your syntax. If it says "Syntax OK", you can safely run `sudo systemctl reload nginx`.

### Decoding Nginx Errors
* **502 Bad Gateway:** Nginx is working, but it can't reach your app. Is your PM2 app actually running? Is it running on the correct port?
* **403 Forbidden:** Nginx doesn't have file permission to read your React files. Check your folder ownership.
* **404 Not Found:** Nginx is looking for a file that doesn't exist. Check your `root` directory path, or ensure you have `try_files` configured for React routing.

### Useful PM2 Commands
* **View live logs:** `pm2 logs`
* **Monitor CPU/memory:** `pm2 monit`
* **Restart your app:** `pm2 restart my-app`

---

## Author Details

* **Name:** Kartikey Mishra
