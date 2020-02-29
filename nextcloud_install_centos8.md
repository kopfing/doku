# Install Nextcloud on LEMP Stack (CentOS 8, Nginx, MariaDB, PHP)

## Prerequisites

### Nginx Web Server
Install and start the server.

```
dnf install -y nginx
systemctl enable --now nginx
```

Add firewall rules to default zone (public) to allow web connections from external.

```
firewall-cmd --permanent --add-service=http --add-service=https
firewall-cmd --reload
```

Make user `nginx` owner of the web directory.

```
chown nginx:nginx -R /usr/share/nginx/html
```

I prefer to have a modular site-setup like in Debian so I create the folder structure

```
mkdir /etc/nginx/sites-available
mkdir /etc/nginx/sites-enabled
```

and move the default server block from `/etc/nginx/nginx.conf` to `/etc/nginx/sites-available/default.conf`. Then to activate this design, the following has to be added to the `http` block in `/etc/nginx/nginx.conf`

```
# Load Virtual Host configs
include /etc/nginx/sites-enabled/*.conf
```

Now it is time to test connectivity by entering the IP address of the server into a web browser.

### MariaDB Database Server

MariaDB is a free and backward compatible replacement of MySQL.

```
dnf install -y mariadb-server mariadb
```

Set the data directory for the MariaDB system tables and future nextcloud database.

```
mysql_install_db --user=mysql --datadir=/media/cloudstorage/clouddb
```

Start the service, run the security script and test login to MariaDB shell.

```
systemctl enable --now mariadb
mysql_secure_installation
mysql -u root -p
```

### PHP-FPM
PHP is a widely-used scripting language especially suited for Web development and can be embedded into HTML.
PHP-FPM is a FastCGI process manager.

Install PHP and related modules and start PHP-FPM

```
dnf install -y php php-mysqlnd php-fpm php-opcache php-gd php-xml php-mbstring
systemctl enable --now php-fpm
```

Change the user and group setting in `/etc/php-fpm.d/www.conf` from apache to nginx.

Also in this file you can find the following line.

```
listen = /run/php-fpm/www.sock
```

This indicates that PHP-FPM is listening on a Unix socket instead of a TCP/IP socket, which is good.

Save and close the file. Reload PHP-FPM for the changes to take effect.

```
systemctl reload php-fpm
```



## Table of References
https://wiki.archlinux.org/index.php/Nextcloud and links
https://www.linuxbabe.com/redhat/install-lemp-nginx-mariadb-php7-rhel-8-centos-8
https://packages.debian.org/de/stretch/nginx-full
https://www.linuxbabe.com/redhat/install-nextcloud-rhel-8-centos-8-nginx-lemp
