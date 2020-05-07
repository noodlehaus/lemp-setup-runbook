## LEMP Server Setup Runbook

Instructions for setting up a basic LEMP stack for application hosting on Ubuntu (LTS 20.04).

### Requirements

- VPS instance (Linode or DO)
- Registered domain name

### Initial Setup

Update your distribution packages and install `fail2ban`.

```
# update packages
apt update
apt upgrade -q -y
```

### Accounts Setup

Create a login for your application, and setup SSH keys for passwordless auth.

```
# add application profile and keys
adduser <username>

# setup ssh keys
mkdir /home/<username>/.ssh
cp <pub_key> /home/<username>/.ssh/authorized_keys
chown -R <username>:<username> /home/<username>/.ssh
chmod 700 /home/<username>/.ssh
chmod 400 /home/<username>/.ssh/authorized_keys
```

Add your application profile to sudoers list.

```
EDITOR=<your editor> visudo
```

Add the following line to your user/groups.

```
<username>  ALL=(ALL:ALL) ALL
```

Update SSH so it doesn't allow password-based auth, and disallow `root` login.

```
vim /etc/ssh/sshd_config
```

Look for the following lines, and update them as indicated.

```
PermitRootLogin no
PasswordAuthentication no
```

Make sure to restart the ssh service after the changes.

```
service restart sshd.service
```

### Network and Access Security

Install `fail2ban` and enable `ufw`

```
apt install fail2ban

# set up firewall rules first
ufw allow 22
ufw allow 80
ufw allow 443
ufw enable
```

### Install LEMP packages

Install git, nginx, php7.4++, mysql, and gearman.

```
# git
apt install git

# nginx
apt install nginx

# mysql + access setup
apt install mysql-server
apt install mysql_secure_installation

# gearman
apt install gearman

# php and deps
apt install php7.4
apt install php7.4-opcache
apt install php7.4-mysql
apt install php7.4-json
apt install php7.4-xml
apt install php7.4-intl
apt install php-imagick
apt install php-gearman

# composer for php dep management
apt install composer
```

### Configure NGINX

[Example NGINX site configuration](./nginx/sites-available/example)

We are going to use the default configuration of nginx, to minimize setup. This
means we will keep the document root, along with the default configuration.

Edit `/etc/nginx/sites-available/default` and make the following changes inside
the `server` block.

```
# disable server tokens
server_tokens off;

# disable directory listings
autoindex off;

# add index.php as a default index file
index index.html index.htm index.php;

# enable php-fpm passthrough
location ~ \.php$ {
  include snippets/fastcgi-php.conf;
  fastcgi_pass unix:/var/run/php/php7.4-fpm.sock;
}
```

### SSL Certificates

Pending.

### Gearman Workers via Systemd

[Example service configuration](./systemd/your-worker.service)

Create your user systemd files path.

```
mkdir -p ~/.config/systemd/user
```

Use the following template for your worker unit.

```
[Unit]
Description=Your custom application worker
Requires=mysql.service

[Service]
User=www-data
Group=www-data
PIDFile=/var/run/your-worker.pid
ExecStart=/usr/bin/php /var/www/workers/your-worker.php > /dev/null 2>/dev/null
Type=Simple
KillMode=process
Restart=always

[Install]
WantedBy=multi-user.target
```

Save this file to any of the following path.

```
/home/<username>/.config/systemd/user/your-worker.service
```

Enable the service.

```
systemctl enable your-worker.service
```

### References

- https://plusbryan.com/my-first-5-minutes-on-a-server-or-essential-security-for-linux-servers
- https://www.linode.com/docs/security/ssl/install-lets-encrypt-to-create-ssl-certificates/
- https://medium.com/@benmorel/creating-a-linux-service-with-systemd-611b5c8b91d6
