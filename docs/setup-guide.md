# Server setup guide

How to build the server for this site from scratch.
Placeholders: `<SERVER_IP>` / `<SERVER_IPV6>` are the server's public IPv4 / IPv6 addresses,
`<USER>` is the admin user name.

## Requirements
- Hetzner Cloud account
- Ubuntu 26.04 LTS, smallest shared x86 plan, public IPv4 and IPv6
- An SSH client (Windows has OpenSSH built in)

## 1. SSH key (local machine)
```
ssh-keygen -t ed25519 -C "<label>"
```
Use a passphrase. Only the public key (`id_ed25519.pub`) is ever shared.

## 2. Hetzner Cloud
1. Add the public key under Security → SSH keys.
2. Create a firewall. Inbound from any IPv4/IPv6: ICMP, TCP 22, TCP 80, TCP 443.
3. Create the server with the SSH key and the firewall attached.

## 3. First login and updates
```
ssh root@<SERVER_IP>
apt update && apt upgrade -y
reboot
```

## 4. Admin user
As root:
```
adduser <USER>
usermod -aG sudo <USER>
install -d -m 700 -o <USER> -g <USER> /home/<USER>/.ssh
install -m 600 -o <USER> -g <USER> /root/.ssh/authorized_keys /home/<USER>/.ssh/
```
Check from a second terminal: `ssh <USER>@<SERVER_IP>`, then `sudo whoami` prints `root`.

## 5. SSH hardening
Keep a working session open while changing SSH settings.

`/etc/ssh/sshd_config.d/00-hardening.conf`:
```
PermitRootLogin no
PasswordAuthentication no
KbdInteractiveAuthentication no
PubkeyAuthentication yes
AllowUsers <USER>
```
```
sudo sshd -t
sudo systemctl restart ssh
sudo sshd -T | grep -Ei '^(permitrootlogin|passwordauthentication|kbdinteractiveauthentication|allowusers)'
```
Check: root login and password login are refused.

## 6. Firewall (ufw)
```
sudo ufw allow OpenSSH
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw enable
sudo ufw status verbose
```

## 7. Automatic security updates
`unattended-upgrades` is enabled by default. Allow reboots when an update requires one.

`/etc/apt/apt.conf.d/52unattended-upgrades-local`:
```
Unattended-Upgrade::Automatic-Reboot "true";
Unattended-Upgrade::Automatic-Reboot-Time "04:00";
```
```
apt-config dump | grep -iE 'Automatic-Reboot|Periodic'
```

## 8. nginx
```
sudo apt install nginx -y
sudo mkdir -p /var/www/csabapiller.com
sudo chown <USER>:<USER> /var/www/csabapiller.com
```
Copy the site from the local repository:
```
scp index.html style.css <USER>@<SERVER_IP>:/var/www/csabapiller.com/
```
`/etc/nginx/sites-available/csabapiller.com`:
```
server {
    listen 80 default_server;
    listen [::]:80 default_server;
    server_name _;

    root /var/www/csabapiller.com;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }
}
```
```
sudo ln -s /etc/nginx/sites-available/csabapiller.com /etc/nginx/sites-enabled/
sudo rm /etc/nginx/sites-enabled/default
sudo nginx -t
sudo systemctl reload nginx
```
Check: `http://<SERVER_IP>` shows the page.

## 9. DNS
For both `csabapiller.com` and `csabapiller.hu`, with no proxying (Cloudflare: DNS only):

| Type | Name | Content |
|---|---|---|
| A | `@` | `<SERVER_IP>` |
| AAAA | `@` | `<SERVER_IPV6>` |
| A | `www` | `<SERVER_IP>` |
| AAAA | `www` | `<SERVER_IPV6>` |

`<SERVER_IPV6>` is shown by `ip -6 addr show scope global`. Check: `nslookup <name>` for all four names.

## 10. nginx for the domain names
Replace `/etc/nginx/sites-available/csabapiller.com` with:
```
# Unknown names and the bare IP address: close the connection without answering.
server {
    listen 80 default_server;
    listen [::]:80 default_server;
    server_name _;
    return 444;
}

# The other three names: redirect permanently to the canonical name.
server {
    listen 80;
    listen [::]:80;
    server_name www.csabapiller.com csabapiller.hu www.csabapiller.hu;
    return 301 http://csabapiller.com$request_uri;
}

# The canonical site.
server {
    listen 80;
    listen [::]:80;
    server_name csabapiller.com;

    root /var/www/csabapiller.com;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }
}
```
```
sudo nginx -t
sudo systemctl reload nginx
```
Check: `curl -I http://csabapiller.com` → 200; the other names → 301 to `http://csabapiller.com/`;
`curl -I http://<SERVER_IP>` → empty reply.

## 11. Certificate (Let's Encrypt)
```
sudo apt install certbot python3-certbot-nginx -y
sudo certbot certonly --nginx -d csabapiller.com -d www.csabapiller.com -d csabapiller.hu -d www.csabapiller.hu
sudo certbot certificates
sudo certbot renew --dry-run
```

## 12. HTTPS in nginx
Replace `/etc/nginx/sites-available/csabapiller.com` with:
```
# ---------- HTTP (port 80) ----------

# Unknown names and the bare IP address: close the connection without answering.
server {
    listen 80 default_server;
    listen [::]:80 default_server;
    server_name _;
    return 444;
}

# Our four names over plain HTTP: redirect to the canonical HTTPS address.
# The redirect is inside "location /" so that certbot's challenge location
# (an exact match, which nginx prefers) still works during renewals.
server {
    listen 80;
    listen [::]:80;
    server_name csabapiller.com www.csabapiller.com csabapiller.hu www.csabapiller.hu;

    location / {
        return 301 https://csabapiller.com$request_uri;
    }
}

# ---------- HTTPS (port 443) ----------

# Unknown names and the bare IP address: refuse the TLS handshake.
server {
    listen 443 ssl default_server;
    listen [::]:443 ssl default_server;
    server_name _;
    ssl_reject_handshake on;
}

# The other three names over HTTPS: redirect to the canonical name.
server {
    listen 443 ssl;
    listen [::]:443 ssl;
    http2 on;
    server_name www.csabapiller.com csabapiller.hu www.csabapiller.hu;

    ssl_certificate     /etc/letsencrypt/live/csabapiller.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/csabapiller.com/privkey.pem;

    return 301 https://csabapiller.com$request_uri;
}

# The canonical site.
server {
    listen 443 ssl;
    listen [::]:443 ssl;
    http2 on;
    server_name csabapiller.com;

    ssl_certificate     /etc/letsencrypt/live/csabapiller.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/csabapiller.com/privkey.pem;

    root /var/www/csabapiller.com;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }
}
```
```
sudo nginx -t
sudo systemctl reload nginx
```
Reload nginx after each certificate renewal with a deploy hook,
`/etc/letsencrypt/renewal-hooks/deploy/reload-nginx.sh`:
```
#!/bin/sh
systemctl reload nginx
```
```
sudo chmod +x /etc/letsencrypt/renewal-hooks/deploy/reload-nginx.sh
sudo certbot renew --dry-run --run-deploy-hooks
```
Check: `https://csabapiller.com` → 200; all other names and `http://` → 301 to
`https://csabapiller.com/`; `https://<SERVER_IP>` → TLS handshake refused.

## 13. Security headers
In `/etc/nginx/nginx.conf`, set the existing line to `server_tokens off;`.

`/etc/nginx/snippets/security-headers.conf`:
```
add_header Strict-Transport-Security "max-age=31536000" always;
add_header Content-Security-Policy "default-src 'self'; frame-ancestors 'none'; base-uri 'self'; form-action 'self'" always;
add_header X-Content-Type-Options "nosniff" always;
add_header X-Frame-Options "DENY" always;
add_header Referrer-Policy "strict-origin-when-cross-origin" always;
add_header Permissions-Policy "camera=(), microphone=(), geolocation=()" always;
```
On a fresh setup, start HSTS at `max-age=86400` and raise it once HTTPS is confirmed working.

Add to both HTTPS server blocks, after `ssl_certificate_key`:
```
    include snippets/security-headers.conf;
```
```
sudo nginx -t
sudo systemctl reload nginx
```
Check: `curl -I https://csabapiller.com` shows the headers and `server: nginx` without a version.
SSL Labs and securityheaders.com: grade A.

## Health check
Re-run every few weeks.
```
sudo ss -tulpn
sudo journalctl -u ssh --since "7 days ago" | grep -E "Accepted (publickey|password)"
awk -F: '$3 == 0 {print $1}' /etc/passwd
getent group sudo
apt list --upgradable
ls /var/run/reboot-required 2>/dev/null && echo "Reboot needed" || echo "No reboot needed"
sudo tail -5 /var/log/unattended-upgrades/unattended-upgrades.log
systemctl --failed
```
Expected: only sshd and nginx listen publicly; logins only by `<USER>` with a public key;
only `root` has UID 0; only `<USER>` is in `sudo`; no pending updates; no failed units.
