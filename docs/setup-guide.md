# Server setup guide

How to build the server for this site from scratch.
Placeholders: `<SERVER_IP>` is the server's public IPv4 address, `<USER>` the admin user name.

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
