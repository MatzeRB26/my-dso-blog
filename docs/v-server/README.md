# V-Server Configuration & Deployment Documentation

This documentation covers the step-by-step setup and configuration of an Ubuntu V-Server hosting a Docusaurus application using NGINX.

---

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [SSH Security & Server Hardening](#ssh-security--server-hardening)
3. [NGINX Web Server Setup](#nginx-web-server-setup)
4. [Git & GitHub SSH Configuration](#git--github-ssh-configuration)
5. [Docusaurus Deployment Process](#docusaurus-deployment-process)
6. [Verification & Testing](#verification--testing)

---

## Architecture Overview

- **Server IP**: `SERVER_IP`
- **OS**: Ubuntu LTS
- **Web Server**: NGINX
- **Runtime**: Node.js 20 LTS / npm
- **Web Root**: `/var/www/docusaurus`

---

## SSH Security & Server Hardening

To secure server access, SSH password-based authentication was disabled in favor of key-based authentication generated locally on the client machine.

1. **Local Key Generation**:
   Key pairs were generated on the client machine (Windows `.ssh` folder):
    ```bash
    ssh-keygen -t ed25519 -C "example@example.com"
    ```
2. **Public Key Transfer**:
   The public key was copied to the server's ~/.ssh/authorized_keys file using ssh-copy-id:
    ```bash
    ssh-copy-id root@SERVER_IP
    ```
    Verify Passwordless Login:
    Before disabling password authentication, a new terminal window was opened to test and verify that key-based authentication worked without a password:
    ```bash
    ssh root@SERVER_IP
    ```

3. **SSH Daemon Configuration**:
   Once key-based login was confirmed working, configured /etc/ssh/sshd_config to disable password authentication and root login:
    ```bash
    PasswordAuthentication no
    PermitRootLogin no
    ```

4. **Service Restart**:
    ```bash
    sudo systemctl restart sshd
    ```

---

## NGINX Web Server Setup

NGINX was installed and configured as the primary web server to serve the static production build of the Docusaurus application.

1. **Installation**:
    ```bash
    sudo apt update
    sudo apt install nginx -y
    ```
2. **Server Block Configuration**:
   Created a new site configuration file at `/etc/nginx/sites-available/docusaurus`:

    ```nginx
    server {
        listen 80;
        server_name SERVER_IP;

        root /var/www/docusaurus;
        index index.html;

        location / {
            try_files $uri $uri/ /index.html;
        }
    }
    ```

3. **Enable Site & Restart NGINX**:
    ```bash
    sudo ln -s /etc/nginx/sites-available/docusaurus /etc/nginx/sites-enabled/
    sudo nginx -t
    sudo systemctl restart nginx
    ```

---

## Git & GitHub SSH Configuration

Configured Git on the V-Server and generated dedicated SSH keypairs for secure interaction with GitHub repositories.

1. **Server SSH Key Generation**:
    ```bash
    ssh-keygen -t ed25519 -C "server-deploy@example.com"
    ```
2. **Add Key to GitHub**:
   Copy the public key content (`cat ~/.ssh/id_ed25519.pub`) and add it under **GitHub Account Settings > SSH and GPG keys**.
3. **Verify Connection**:
    ```bash
    ssh -T git@github.com
    ```

---

## Docusaurus Deployment Process

The deployment workflow involves pulling the repository, installing dependencies, building static files, and placing them into the web root directory.

1. **Directory Setup**:
    ```bash
    sudo mkdir -p /var/www/docusaurus
    sudo chown -R $USER:$USER /var/www/docusaurus
    ```
2. **Repository Clone / Pull**:
    ```bash
    git clone git@github.com:<your-username>/<your-repo>.git /tmp/docusaurus-repo
    cd /tmp/docusaurus-repo
    ```
3. **Build Application**:
    ```bash
    npm install
    npm run build
    ```
4. **Deploy Build Artifacts**:
    ```bash
    cp -r build/* /var/www/docusaurus/
    ```

---

## Verification & Testing

Ensured the web application and server services are functioning correctly.

1. **Service Status Check**:
    ```bash
    sudo systemctl status nginx
    ```
2. **Port & HTTP Response Test**:
    ```bash
    curl -I http://SERVER_IP
    ```
3. **Browser Verification**:
   Navigate to `SERVER_IP` in a web browser to confirm that the Docusaurus blog loads properly.
