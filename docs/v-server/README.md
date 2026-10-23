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
* **Server IP**: `188.245.242.210`
* **OS**: Ubuntu LTS
* **Web Server**: NGINX
* **Runtime**: Node.js 20 LTS / npm
* **Web Root**: `/var/www/docusaurus`

---

## SSH Security & Server Hardening
To secure server access, SSH password-based authentication was disabled in favor of key-based authentication generated locally on the client machine.

1. **Local Key Generation**:
   Key pairs were generated on the client machine (Windows `.ssh` folder):
   ```bash
   ssh-keygen -t ed25519 -C "m.stojak14@gmail.com"