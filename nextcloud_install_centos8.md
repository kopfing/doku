# Install Nextcloud on LEMP Stack (CentOS 8, Nginx, MariaDB, PHP)

Installation instructions as done on a clean headless VM with CentOS 8 installed.

All commands are execeuted as root.

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
        add_header Cache-Control "public, max-age=7200";
        # Add headers to serve security related headers (It is intended to
        # have those duplicated to the ones above)
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


- deactivate default site (remove symlink or forbidden?)

## Table of References
https://wiki.archlinux.org/index.php/Nextcloud and links

https://www.linuxbabe.com/redhat/install-lemp-nginx-mariadb-php7-rhel-8-centos-8

https://packages.debian.org/de/stretch/nginx-full

https://www.linuxbabe.com/redhat/install-nextcloud-rhel-8-centos-8-nginx-lemp
