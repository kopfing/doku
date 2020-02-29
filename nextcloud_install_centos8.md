# Install Nextcloud on LEMP Stack (CentOS 8, Nginx, MariaDB, PHP)

Installation instructions as done on a clean headless VM with CentOS 8 installed.

All commands are execeuted as root.

## Prerequisites

First install some dependencies needed during installation and in every day use situations.

```
dnf install -y epel-release yum-utils unzip curl wget bash-completion policycoreutils-python-utils mlocate bzip2
dnf update -y
```

Resolve dependency fail, this is only needed if the command `dnf list` shows "Modular dependency problems" with perl.

```
dnf install -y perl
yum module enable perl:5.26
```

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

and move the default server block from `/etc/nginx/nginx.conf` to `/etc/nginx/sites-available/default.conf`. Then to activate this design, the following has to be added to the `http` block in `/etc/nginx/nginx.conf`.

```
# Load Virtual Host configs
include /etc/nginx/sites-enabled/*.conf
```

And the `default.conf` has to be symlinked to the `sites-enabled` folder.

```
cd /etc/nginx/sites-enabled
ln -s /etc/nginx/sites-available/default.conf
```

Now it is time to test connectivity by entering the IP address of the server into a web browser.

### MariaDB Database Server

MariaDB is a free and backward compatible replacement of MySQL.

```
dnf install -y mariadb-server mariadb
```

Set the data directory for the MariaDB system tables and future nextcloud database in `/etc/my.cnf.d/mariadb-server.cnf`

Then label the new data dir as suitable for MariaDB data files for SELinux.

```
ls -alZ /media/cloudstorage/cloudb
semanage fcontext -a -t mysqld_db_t "/media/cloudstorage/clouddb(/.*)?"
restorecon -R -v /media/cloudstorage/clouddb
ls -alZ /media/cloudstorage/clouddb
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

Save and close the file, then open `/etc/php.ini` for editing and locate the settin `cgi.fix_pathinfo` which must be uncommented and set to `0`.

Reload PHP-FPM for the changes to take effect.

```
systemctl reload php-fpm
```

#### Test PHP

By default, the Nginx package on CentOS 8 includes configurations for PHP-FPM (`/etc/nginx/conf.d/php-fpm.conf` and `/etc/nginx/default.d/php.conf`). To test PHP-FPM with Nginx Web server, we need to create a `info.php` file in the document root directory

```
vim /usr/share/nginx/html/info.php
```

with following content

```
<?php phpinfo(); ?>
```

Now in the browser enter `server-ip-address/info.php`. If the browser fails to display the PHP info but prompts you to download the `info.php` file, restart nginx and PHP-FPM.

```
sudo systemctl restart nginx php-fpm
```

For security reasons, the `info.php` should be deleted again, however this info page will be helpful throughout the process of setting up nextcloud because the php modules can always be checked for availability. The "default" site will be deactivated at the end of this How-To.

## Download NextCloud

Download the NextCloud archive to the server, it can be found under https://nextcloud.com/install/. Change the version number accordingly.

```
cd ~
curl https://download.nextcloud.com/server/releases/nextcloud-18.0.1.tar.bz2 -o nextcloud.tar.bz2
tar xfvj nextcloud.tar.bz2
```

Move the folder to the web root directory and change ownership.

```
mv nextcloud /usr/share/nginx/
chown -R nginx:nginx /usr/share/nginx/nextcloud/
```

## Create Database and User in MariaDB

Log into MariaDB database server

```
mysql -u root -p
```

Create database and user with privileges on the database.

```
create database nextcloud default character set 'utf8' collate 'utf8_unicode_ci';
grant all privileges on nextcloud.* to 'nextcloud'@'localhost' identified by 'password';
flush privileges;
```

Then logout from the DB server by pressing `Ctrl+D`.

## Create a nginx Config File for NextCloud

```
vim /etc/nginx/sites-available/nextcloud.conf
```

Put the following text into the file and replace the `server_name` with the actual FQDN.

```
server {
    listen 80;
    server_name nextcloud.your-domain.com;

    # Add headers to serve security related headers
    add_header Strict-Transport-Security "max-age=15768000; includeSubDomains;" always;
    add_header X-Content-Type-Options nosniff;
    add_header X-XSS-Protection "1; mode=block";
    add_header X-Robots-Tag none;
    add_header X-Download-Options noopen;
    add_header X-Permitted-Cross-Domain-Policies none;
    add_header Referrer-Policy no-referrer;

    #I found this header is needed on Debian/Ubuntu/CentOS/RHEL, but not on Arch Linux.
    add_header X-Frame-Options "SAMEORIGIN";

    # Path to the root of your installation
    root /usr/share/nginx/nextcloud/;

    access_log /var/log/nginx/nextcloud.access;
    error_log /var/log/nginx/nextcloud.error;

    location = /robots.txt {
        allow all;
        log_not_found off;
        access_log off;
    }

    # The following 2 rules are only needed for the user_webfinger app.
    # Uncomment it if you're planning to use this app.
    #rewrite ^/.well-known/host-meta /public.php?service=host-meta last;
    #rewrite ^/.well-known/host-meta.json /public.php?service=host-meta-json
    # last;

    location = /.well-known/carddav {
        return 301 $scheme://$host/remote.php/dav;
    }
    location = /.well-known/caldav {
       return 301 $scheme://$host/remote.php/dav;
    }

    location ~ /.well-known/acme-challenge {
      allow all;
    }

    # set max upload size
    client_max_body_size 512M;
    fastcgi_buffers 64 4K;

    # Disable gzip to avoid the removal of the ETag header
    gzip off;

    # Uncomment if your server is build with the ngx_pagespeed module
    # This module is currently not supported.
    #pagespeed off;

    error_page 403 /core/templates/403.php;
    error_page 404 /core/templates/404.php;

    location / {
       rewrite ^ /index.php$uri;
    }

    location ~ ^/(?:build|tests|config|lib|3rdparty|templates|data)/ {
       deny all;
    }
    location ~ ^/(?:\.|autotest|occ|issue|indie|db_|console) {
       deny all;
     }

    location ~ ^/(?:index|remote|public|cron|core/ajax/update|status|ocs/v[12]|updater/.+|ocs-provider/.+|core/templates/40[34])\.php(?:$|/) {
       include fastcgi_params;
       fastcgi_split_path_info ^(.+\.php)(/.*)$;
       fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
       fastcgi_param PATH_INFO $fastcgi_path_info;
       #Avoid sending the security headers twice
       fastcgi_param modHeadersAvailable true;
       fastcgi_param front_controller_active true;
       fastcgi_pass unix:/run/php-fpm/www.sock;
       fastcgi_intercept_errors on;
       fastcgi_request_buffering off;
    }

    location ~ ^/(?:updater|ocs-provider)(?:$|/) {
       try_files $uri/ =404;
       index index.php;
    }

    # Adding the cache control header for js and css files
    # Make sure it is BELOW the PHP block
    location ~* \.(?:css|js)$ {
        try_files $uri /index.php$uri$is_args$args;
        add_header Cache-Control "public, max-age=15778463";
        # Add headers to serve security related headers (It is intended to
        # have those duplicated to the ones above)
        add_header Strict-Transport-Security "max-age=15768000; includeSubDomains;" always;
        add_header X-Content-Type-Options nosniff;
        add_header X-XSS-Protection "1; mode=block";
        add_header X-Robots-Tag none;
        add_header X-Download-Options noopen;
        add_header X-Permitted-Cross-Domain-Policies none;
        # Optional: Don't log access to assets
        access_log off;
   }

   location ~* \.(?:svg|gif|png|html|ttf|woff|ico|jpg|jpeg)$ {
        try_files $uri /index.php$uri$is_args$args;
        # Optional: Don't log access to other assets
        access_log off;
   }
}
```

Then test the nginx configuration.

```
nginx -t
```

And if successful, reload nginx.

## Install and Enable PHP Modules

Install PHP modules and build imagick.

```
dnf install -y php-common php-json php-curl php-zip php-bz2 php-intl php-pecl-apcu php-pear gcc curl-devel php-devel zlib-devel pcre-devel make
yum config-manager --set-enabled PowerTools
dnf install -y ImageMagick ImageMagick-devel
pecl install imagick
```

Load the extension in the php.ini file with

```
extension=imagick.so
```

Tell SELinux to allow PHP-FPM to use execmem.

```
setsebool -P httpd_execmem 1
```

The option `-P` means that the setting should be permanent.

Reload PHP-FPM.

## Setting up Permissions

Tell SELinux to allow nginx and PHP-FPM to read and write to the nextcloud webroot tree.

```
chcon -t httpd_sys_rw_content_t /usr/share/nginx/nextcloud -R
```

Tell SELinux to allow nginx to make network requests to other server. This is needed to request TLS certificate status from Let's Encrypt CA server.

```
setsebool -P httpd_can_network_connect 1
```

Give the nginx user permissions to read and write to the following 3 directories with `setfacl`.

```
setfacl -R -m u:nginx:rwx /var/lib/php/opcache/
setfacl -R -m u:nginx:rwx /var/lib/php/session/
setfacl -R -m u:nginx:rwx /var/lib/php/wsdlcache/
```

Don't change the owner of these files for possible future use of apache web server.

## Enable https

To request a certificate from Let's Encrypt and install it to the nginx configuration use certbot and its nginx plugin

```
dnf install -y certbot python3-certbot
pip3 install certbot-nginx
certbot --nginx
```

Go through the script and restart nginx.

## NextCloud Install Wizard

Now the install wizard is accessible using https.

Because it is unsafe to store NextCloud users data in the NextCloud root, create a folder for the data and set permissions.

```
mkdir /media/cloudstorage/clouddata
chown nginx:nginx -R /media/cloudstorage/clouddata
chcon -t httpd_sys_rw_content_t /media/cloudstorage/clouddata/ -R
```

Now navigate to your NextCloud server in the browser and complete Setup.

## Post Install

### Setup Email Notifications
For example for password-resetting mails.

Log into NextCloud with the administrator account and navigate to **Settings -> Basic**. Select the send mode `smtp` and fill out the details.

Tell SELinux to allow nginx to send mail.

```
setsebool -P httpd_can_sendmail on
```

### Performance and Usability

In `/etc/nginx/sites-available/nextcloud.conf` the maximum file size is already set to 512M. This can be further increased if needed.

PHP also has to be configured accordingly. Set the line

```
upload_max_filesize = 2M
```

in `/etc/php.ini` to fit your nginx configuration and restart PHP-FPM.

### Automating Certificate Renewal
Edit root user's crontab file

```
crontab -e
```

Add the following line at the end of the file to run the Cron job daily. If the certificate is going to expire in 30 days, certbot will try to renew the certificate. It’s necessary to reload the Nginx service to pick up new certificate and key file.

```
@daily certbot renew --quiet && systemctl reload nginx
```


### Security

### Last Checks if Everything is Setup
Log into NextCloud with the administrator account and navigate to **Settings -> Overview**. Follow the instructions and tips.

Go to **Settings -> Logging** and check for any errors.



- deactivate default site (remove symlink or forbidden?)
- performance and extensions
  - https://docs.nextcloud.com/server/latest/admin_manual/installation/example_centos.html#manually-building-redis-imagick-optional
- backup einrichten

## Table of References
https://wiki.archlinux.org/index.php/Nextcloud and links

https://www.linuxbabe.com/redhat/install-lemp-nginx-mariadb-php7-rhel-8-centos-8

https://packages.debian.org/de/stretch/nginx-full

https://www.linuxbabe.com/redhat/install-nextcloud-rhel-8-centos-8-nginx-lemp

https://docs.nextcloud.com/server/latest/admin_manual/installation/example_centos.html

https://centosfaq.org/centos/yum-dnf-possible-confusion-centos-8/

https://certbot.eff.org/lets-encrypt/centosrhel7-nginx.html

https://stackoverflow.com/questions/26474222/mariadb-10-centos-7-moving-datadir-woes

https://docs.nextcloud.com/server/18/admin_manual/installation/harden_server.html
