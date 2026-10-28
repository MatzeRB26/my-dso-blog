# V-Server Setup

This page documents how I set up the virtual server for the DevSecOps V-Server project: key based SSH access, a disabled password login, NGINX serving a custom HTML page on port 8081 instead of the packaged welcome screen, and a git configuration that can talk to GitHub from the server itself.

The machine runs Ubuntu 24.04.5 LTS. In all commands, values in angle brackets are placeholders that have to be replaced with your own values.

## Table of Contents
- [Architecture overview](#architecture-overview)
- [Prerequisites](#prerequisites)
- [Quickstart](#quickstart)
- [Usage](#usage)
  * [Generating the key pair](#generating-the-key-pair)
  * [Deploying the public key](#deploying-the-public-key)
  * [Disabling the password login](#disabling-the-password-login)
  * [NGINX with a custom entry point](#nginx-with-a-custom-entry-point)
  * [Git configuration](#git-configuration)
  * [GitHub access from the server](#github-access-from-the-server)
  * [Operating the server](#operating-the-server)
- [Verification](#verification)
- [Security notes](#security-notes)
- [Checklist](#checklist)
- [Further References](#further-references)

import GithubLinkAdmonition from '@site/src/components/GithubLinkAdmonition';

<GithubLinkAdmonition 
    link="https://github.com/MatzeRB26/my-dso-blog"
    title="Github Tip" 
    type="tip"
/>

Checkout this repository to see the configuration files described below.

## Architecture overview

| Component | Value |
| :--- | :--- |
| Server address | `<SERVER_IP>` |
| Operating system | Ubuntu 24.04.5 LTS |
| Web server | NGINX, site configuration `/etc/nginx/sites-available/v-server` |
| Custom page | `/var/www/v-server/index.html`, served on port 8081 |
| Access | SSH with key pair only, password login disabled |

## Prerequisites
- A virtual server running a current Ubuntu, reachable over the network, and an account on it that is a member of the `sudo` group.
- Its IP address (`<SERVER_IP>`) and the user name of that account (`<SERVER_USER>`). A password is needed exactly once, to deploy the first public key; everything after that runs over the key.
- An SSH client on the workstation. `ssh`, `ssh-keygen`, `ssh-copy-id` and `scp` ship with Linux and macOS, and on Windows with Git for Windows.
- A GitHub account, if the server is to interact with repositories.
- Your own `index.html`, the page that NGINX will serve.

## Quickstart
Eight steps from a fresh server to a hardened machine with a custom web page. Each step is explained in the [Usage](#usage) section.

1. Generate a key pair on your own machine, never on the server:
   ```bash
   ssh-keygen -t ed25519 -f ~/.ssh/vserver_da -C "v-server"
   ```
2. Copy the public key to the server:
   ```bash
   ssh-copy-id -i ~/.ssh/vserver_da.pub <SERVER_USER>@<SERVER_IP>
   ```
3. Confirm that the key based login works without a password:
   ```bash
   ssh -o PasswordAuthentication=no <SERVER_USER>@<SERVER_IP> 'echo ok'
   ```
4. Only after `ok` appeared: switch the password login off and restart the SSH daemon:
   ```bash
   ssh <SERVER_USER>@<SERVER_IP>
   sudo tee /etc/ssh/sshd_config.d/99-hardening.conf <<'EOF'
   PubkeyAuthentication yes
   PasswordAuthentication no
   KbdInteractiveAuthentication no
   PermitRootLogin prohibit-password
   EOF
   sudo sshd -t && sudo systemctl restart ssh
   ```
5. Install NGINX and publish your page on port 8081:
   ```bash
   sudo apt-get update && sudo apt-get install -y nginx
   sudo mkdir -p /var/www/v-server
   sudo cp ~/index.html /var/www/v-server/index.html
   sudo tee /etc/nginx/sites-available/v-server <<'EOF'
   server {
       listen 8081;
       listen [::]:8081;
       server_name _;
       root /var/www/v-server;
       index index.html;
       location / {
           try_files $uri $uri/ =404;
       }
       server_tokens off;
       access_log /var/log/nginx/v-server.access.log;
       error_log  /var/log/nginx/v-server.error.log;
   }
   EOF
   sudo ln -s /etc/nginx/sites-available/v-server /etc/nginx/sites-enabled/v-server
   sudo nginx -t && sudo systemctl reload nginx
   ```
   The `index.html` has to be on the server first, for example with `scp index.html <SERVER_USER>@<SERVER_IP>:~/` from the workstation.
6. Install git and set the identity:
   ```bash
   sudo apt-get install -y git
   git config --global user.name "<YOUR_GITHUB_USERNAME>"
   git config --global user.email "<YOUR_EMAIL>"
   ```
7. Give the server its own key pair for GitHub and register the public key in GitHub under *Settings, SSH and GPG keys*:
   ```bash
   ssh-keygen -t ed25519 -f ~/.ssh/github_vserver -C "v-server github"
   cat ~/.ssh/github_vserver.pub
   ```
8. Check the result:
   ```bash
   curl -I http://<SERVER_IP>:8081/
   ssh -T git@github.com
   ```

## Usage
This section explains the steps of the quickstart in detail.

### Generating the key pair
The key pair belongs to the workstation, so it is created there. A key generated on the server would have to travel to the client at some point, which defeats the purpose of asymmetric authentication.

```bash
ssh-keygen -t ed25519 -f ~/.ssh/vserver_da -C "v-server"
```

`ed25519` is the current default: short keys, fast verification, and no parameter choices that can be made badly. The private key stays in `~/.ssh/vserver_da`, the public key in `~/.ssh/vserver_da.pub`. On Windows, the `.ssh` folder of the user profile is the equivalent location.

A matching entry in `~/.ssh/config` makes every later command shorter and pins the identity, so the SSH client does not offer unrelated keys to the server:

```text
Host <SERVER_IP>
    HostName <SERVER_IP>
    User <SERVER_USER>
    IdentityFile ~/.ssh/vserver_da
    IdentitiesOnly yes
```

With this entry, `ssh <SERVER_IP>` is enough to connect.

### Deploying the public key
`ssh-copy-id` appends the public key to `~/.ssh/authorized_keys` on the server and fixes the permissions on the way:

```bash
ssh-copy-id -i ~/.ssh/vserver_da.pub <SERVER_USER>@<SERVER_IP>
```

This is the one step that still needs the password, because it is the last moment at which the server has no other way to recognise the client. If a shell on the server is already open, the same result can be achieved by appending the public key by hand:

```bash
mkdir -p ~/.ssh && chmod 700 ~/.ssh
echo 'ssh-ed25519 AAAA... v-server' >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
```

The permissions matter: `sshd` ignores `authorized_keys` when the file or its directory is writable by anyone but the owner.

### Disabling the password login
The login with the key has to be verified before passwords are switched off, otherwise a mistake in the configuration locks the account out of the machine. The check is one command, and it has to succeed without asking for anything. It is best run from a second terminal window while the first session stays open:

```bash
ssh -o PasswordAuthentication=no <SERVER_USER>@<SERVER_IP> 'echo ok'
```

```text
ok
```

`PasswordAuthentication=no` is what makes this a real test rather than a habit: it tells the client not to fall back to a password prompt, so the command can only succeed through the key. A password prompt instead of `ok` means the key is not in place, and the rest of this section has to wait.

Ubuntu reads `/etc/ssh/sshd_config.d/*.conf` from an `Include` at the very top of `sshd_config`, and `sshd` keeps the first value it sees for an option. A drop-in file therefore overrides the defaults without editing the packaged configuration, which keeps later package upgrades free of conflicts:

```text
# /etc/ssh/sshd_config.d/99-hardening.conf
PubkeyAuthentication yes
PasswordAuthentication no
KbdInteractiveAuthentication no
PermitRootLogin prohibit-password
```

`KbdInteractiveAuthentication` is disabled as well. Left on, PAM can still offer an interactive password dialogue, and the password login would only appear to be closed.

The syntax is checked before the daemon is restarted. `sshd -T` prints the configuration as it will actually be applied, which is the more useful of the two checks:

```bash
sudo sshd -t
sudo sshd -T | grep -E "passwordauthentication|pubkeyauthentication"
sudo systemctl restart ssh
```

Restarting the daemon does not drop existing sessions, as each connection is served by its own process.

### NGINX with a custom entry point
```bash
sudo apt-get update && sudo apt-get install -y nginx
```

The package ships a server block on port 80 that serves the welcome page. It stays untouched. The project gets a site configuration of its own that listens on port 8081 and serves the custom page:

```text
# /etc/nginx/sites-available/v-server
server {
    listen 8081;
    listen [::]:8081;

    server_name _;

    root /var/www/v-server;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }

    server_tokens off;

    access_log /var/log/nginx/v-server.access.log;
    error_log  /var/log/nginx/v-server.error.log;
}
```

`root` points at the directory with the page and `index` names the file that is answered for `/`. `try_files` returns a 404 for everything that does not exist instead of falling back to another page.

`server_tokens off` removes the exact version number from error pages and from the `Server` header. That is not a security measure in itself, but there is no reason to tell every visitor which patch level is running.

```bash
sudo mkdir -p /var/www/v-server
sudo cp index.html /var/www/v-server/index.html
sudo ln -s /etc/nginx/sites-available/v-server /etc/nginx/sites-enabled/v-server
sudo nginx -t
sudo systemctl reload nginx
```

`nginx -t` validates the configuration before it is applied; `reload` then replaces the worker processes without dropping connections.

### Git configuration
Most Ubuntu server images carry git already, but not all of them, so it is installed before it is configured. `apt-get` leaves an existing installation untouched, which makes the command safe to run either way:

```bash
sudo apt-get update && sudo apt-get install -y git
git --version
```

The identity on the server matches the one on the workstation and the one GitHub knows, so commits made on the server are attributed to the same account:

```bash
git config --global user.name "<YOUR_GITHUB_USERNAME>"
git config --global user.email "<YOUR_EMAIL>"
git config --global init.defaultBranch main
```

### GitHub access from the server
The server gets a key pair of its own instead of a copy of the workstation key. A key that exists in one place only can be withdrawn in one place only, which is what makes a lost machine a contained problem:

```bash
ssh-keygen -t ed25519 -f ~/.ssh/github_vserver -C "v-server github"
```

The public key (`cat ~/.ssh/github_vserver.pub`) is registered under *Settings, SSH and GPG keys* on GitHub. An entry in the server's `~/.ssh/config` points git at that key:

```text
Host github.com
    HostName github.com
    User git
    IdentityFile ~/.ssh/github_vserver
    IdentitiesOnly yes
```

The connection is then confirmed with `ssh -T git@github.com`.

### Operating the server
What lives where on the machine:

| Path | Contents |
| :--- | :--- |
| `/var/www/v-server/index.html` | the custom page, served on port 8081 |
| `/etc/nginx/sites-available/v-server` | the site configuration, symlinked into `sites-enabled` |
| `/etc/ssh/sshd_config.d/99-hardening.conf` | the drop-in that switches password logins off |
| `/var/log/nginx/v-server.*.log` | access and error log of this site |

Changing the page means copying a new file over it and nothing else, as NGINX reads it per request:

```bash
scp index.html <SERVER_USER>@<SERVER_IP>:/tmp/
ssh <SERVER_USER>@<SERVER_IP> 'sudo cp /tmp/index.html /var/www/v-server/index.html'
```

Changing the site configuration needs a test and a reload. `reload` replaces the worker processes without dropping connections, which `restart` would:

```bash
sudo nginx -t && sudo systemctl reload nginx
```

## Verification
The services are running:

```bash
sudo systemctl status nginx
sudo systemctl status ssh
```

The key based login works:

```bash
ssh <SERVER_IP> 'echo ok'
```

The password login is refused. Public key authentication is switched off for this test, so the client has nothing else to offer:

```bash
ssh -o PubkeyAuthentication=no <SERVER_USER>@<SERVER_IP>
```

```text
<SERVER_USER>@<SERVER_IP>: Permission denied (publickey).
```

The list in the parentheses is the point: it names the methods the server still accepts. As long as `password` is missing from it, the password login is closed.

The web server answers with the custom page on port 8081 rather than the NGINX welcome screen:

```bash
curl -I http://<SERVER_IP>:8081/
```

```text
HTTP/1.1 200 OK
Server: nginx
```

In the browser, `http://<SERVER_IP>:8081/` shows the page. The connection from the server to GitHub is confirmed with `ssh -T git@github.com`.

## Security notes
* No private key, no password and no host specific secret is part of this repository. Only public keys, configuration files and commands are documented here.
* The workstation key was created without a passphrase so that the setup could be automated. A key that protects something worth protecting should carry one and be unlocked through `ssh-agent` once per session.
* The firewall (`ufw`) is inactive on this machine. Only SSH and NGINX listen on a public port, so nothing is exposed that should not be, but a rule set limited to ports 22 and 8081 would be the obvious next step. A firewall of the hosting provider has to allow port 8081 as well, otherwise the page is not reachable from outside.

## Checklist
The project checklist is part of this repository: [Git + V-Server Checkliste](./Git%20+%20VServer%20Checkliste.pdf).

## Further References
* [OpenSSH manual: ssh](https://man.openbsd.org/ssh)
* Ubuntu Server documentation: OpenSSH
* [NGINX: Serving static content](https://nginx.org/en/docs/beginners_guide.html#static)
* [GitHub Docs: Connecting to GitHub with SSH](https://docs.github.com/en/authentication/connecting-to-github-with-ssh)
