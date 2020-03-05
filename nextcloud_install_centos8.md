# Nextcloud on LEMP Stack (CentOS 8, Nginx, MariaDB, PHP)

Installation instructions as tested for Nextcloud 18 on a headless VM with a clean CentOS 8 install.

All commands are execeuted as root.

- [Prerequisites](#prerequisites)
  - [Nginx Web Server](#nginx-web-server)
  - [MariaDB Database Server](#mariadb-database-server)
  - [PHP-FPM](#php-fpm)
    - [Test PHP](#test-php)
- [Download Nextcloud](#download-nextcloud)
- [Create a Database and User in MariaDB](#create-database-and-user-in-mariadb)
- [Create a nginx Config File for Nextcloud](#create-database-and-user-in-mariadb)
- [Install and Enable PHP Modules](#install-and-enable-php-modules)
- [Setting up Permissions](#setting-up-permissions)
- [Enable https](#enable-https)
  - [Automating Certificate Renewal](#automating-certificate-renewal)
- [Nextcloud Install Wizard](#nextcloud-install-wizard)
- [Post Install](#post-install)
  - [Setup Email Notifications](#setup-email-notifications)
  - [Performance and Usability](#performance-and-usability)
    - [Maximum Upload Size](#maximum-upload-size)
    - [Memory Caching](#memory-caching)
    - [Tuning the Database](#tuning-the-database)
    - [Tuning PHP](#tuning-php)
  - [Background Jobs](#background-jobs)
  - [Security](#security)
  - [Last Checks if Everthing is Setup](#last-checks-if-everything-is-setup)
- [Tips](#tips)
- [Table of References](#table-of-references)

## Prerequisites

First install some dependencies needed during installation and in every day use situations.

```
dnf install -y epel-release yum-utils unzip curl wget bash-completion policycoreutils-python-utils mlocate bzip2 git
dnf update -y
```

Resolve dependency fail, this is only needed if the command `dnf list` shows "Modular dependency problems" with perl.

```
dnf install -y perl
yum module enable perl:5.26
```

Now the LEMP stack will be installed.

### Nginx Web Server
Nginx (pronounced "Engine X") is a high performance web server and reverse proxy server.

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

and move the default server block from `/etc/nginx/nginx.conf` to `/etc/nginx/sites-available/default.conf`.<br/>
Then to activate this design, the following has to be added to the `http` block in `/etc/nginx/nginx.conf`.

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
PHP is a widely-used scripting language especially suited for Web development and can be embedded into HTML.<br/>
PHP-FPM is a FastCGI process manager which can manage an always running stack of PHP processes for faster execution of php scripts.

Install PHP and related modules and start PHP-FPM

```
dnf install -y php php-mysqlnd php-fpm php-opcache php-gd php-xml php-mbstring
systemctl enable --now php-fpm
```

Change the user and group setting in `/etc/php-fpm.d/www.conf` from apache to nginx.<br/>
Also in this file you can find the following line.

```
listen = /run/php-fpm/www.sock
```

This indicates that PHP-FPM is listening on a Unix socket instead of a TCP/IP socket, which is good.<br/>
Save and close the file, then open `/etc/php.ini` for editing and locate the setting `cgi.fix_pathinfo` which must be uncommented and set to `0`.

Reload PHP-FPM for the changes to take effect.

```
systemctl reload php-fpm
```

#### Test PHP

By default, the Nginx package on CentOS 8 includes configurations for PHP-FPM (`/etc/nginx/conf.d/php-fpm.conf` and `/etc/nginx/default.d/php.conf`).<br/>
To test PHP-FPM with Nginx Web server, we need to create a `info.php` file in the document root directory

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

## Download Nextcloud

Download the Nextcloud archive to the server, it can be found under https://nextcloud.com/install/. Change the version number accordingly.

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

## Create a nginx Config File for Nextcloud

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

The extensions are loaded automatically by installing a config file into `/etc/php.d/`.<br/>
Just imagick has to be loaded by adding following line to `/etc/php.ini`

```
extension=imagick.so
```

Tell SELinux to allow PHP-FPM to use execmem.

```
setsebool -P httpd_execmem 1
```

The option `-P` means that the setting should be permanent.

To populate envirenment variable for php-fpm uncomment following lines in `/etc/php-fpm.d/www.conf`.

```
;env[HOSTNAME] = $HOSTNAME
;env[PATH] = /usr/local/bin:/usr/bin:/bin
;env[TMP] = /tmp
;env[TMPDIR] = /tmp
;env[TEMP] = /tmp
```

Reload PHP-FPM.

## Setting up Permissions

Tell SELinux to allow nginx and PHP-FPM to read and write to the nextcloud webroot tree.

```
chcon -t httpd_sys_rw_content_t /usr/share/nginx/nextcloud -R
```

Tell SELinux to allow nginx to make network requests to other server. This is needed to request TLS certificate status from Let's Encrypt CA server.<br/>
Additionally make sure SELinux will not interfere with any operations needed for Nextcloud. If any executed installation step differs from this guide, see the official *SELinux configuration* guide by Nextcloud.

```
semanage fcontext -a -t httpd_sys_rw_content_t '/usr/share/nginx/nextcloud/data(/.*)?'
semanage fcontext -a -t httpd_sys_rw_content_t '/usr/share/nginx/nextcloud/config(/.*)?'
semanage fcontext -a -t httpd_sys_rw_content_t '/usr/share/nginx/nextcloud/apps(/.*)?'
semanage fcontext -a -t httpd_sys_rw_content_t '/usr/share/nginx/nextcloud/.htaccess'
semanage fcontext -a -t httpd_sys_rw_content_t '/usr/share/nginx/nextcloud/.user.ini'
semanage fcontext -a -t httpd_sys_rw_content_t '/usr/share/nginx/nextcloud/3rdparty/aws/aws-sdk-php/src/data/logs(/.*)?'
restorecon -R '/usr/share/nginx/nextcloud'
setsebool -P httpd_can_network_connect 1
```

Give the nginx user permissions to read and write to the following 3 directories with `setfacl`.

```
setfacl -R -m u:nginx:rwx /var/lib/php/opcache/
setfacl -R -m u:nginx:rwx /var/lib/php/session/
setfacl -R -m u:nginx:rwx /var/lib/php/wsdlcache/
```

Don't change the owner of these files for possible future use of apache web server.<br/>
The ACLs for the directories can be checked with `getfacl`.

## Enable https

To request a certificate from Let's Encrypt and install it to the nginx configuration use certbot and its nginx plugin

```
dnf install -y certbot python3-certbot
pip3 install certbot-nginx
certbot --nginx
```

Go through the script and restart nginx.

### Automating Certificate Renewal
Edit root user's crontab file

```
crontab -e
```

Add the following line at the end of the file to run the cron job daily. If the certificate is going to expire in 30 days, certbot will try to renew the certificate. It’s necessary to reload the Nginx service to pick up new certificate and key file.

```
@daily certbot renew --quiet && systemctl reload nginx
```

## Nextcloud Install Wizard

Now the install wizard is accessible using https.

Because it is unsafe to store Nextcloud users data in the Nextcloud root, create a folder for the data and set permissions.

```
mkdir /media/cloudstorage/clouddata
chown nginx:nginx -R /media/cloudstorage/clouddata
chcon -t httpd_sys_rw_content_t /media/cloudstorage/clouddata/ -R
```

Navigate to your Nextcloud server in the browser and complete Setup.

## Post Install

### Setup Email Notifications
For example for password-resetting mails.

Log into Nextcloud with the administrator account and navigate to **Settings -> Basic**. Select the send mode `smtp` and fill out the details.

Tell SELinux to allow nginx to send mail.

```
setsebool -P httpd_can_sendmail 1
```

### Performance and Usability

#### Maximum Upload Size
In `/etc/nginx/sites-available/nextcloud.conf` the maximum file size is already set to 512M. This can be further increased if needed.

PHP also has to be configured accordingly. Set the lines

```
upload_max_filesize
post_max_size
```

in `/etc/php.ini` to fit your nginx configuration and restart PHP-FPM.

#### Memory Caching
PHP opcache, which stores compiled PHP scripts in memory is already installed with the extension `php-opcache`
To significantly improve the server performance enable local and distributed data caching as well as transactional file locking by installing and enabling `APCu` and `Redis`.

Manually build Redis

```
pecl install redis
```

Create a new ini file to make sure, the extension is loaded after the json extension

```
vim /etc/php.d/99-redis.ini
```

and add the following line.

```
extension=redis.so
```

Install and start the Redis service.

```
dnf install -y redis
systemctl enable --now redis
```

To make Redis listen on a Unix socket instead of a TCP socket, open `/etc/redis.conf` and configure following attributes.

```
port 0
unixsocket /var/run/redis/redis.sock
unixsocketperm 770
```

Add the user *nginx* to the group *redis*.

```
gpasswd -a nginx redis
```

Tell SELinux to allow deamons to enable cluster mode.

```
setsebool -P daemons_enable_cluster_mode 1
```

APCu is already installed, so now both extensions must be enabled by adding following lines to<br/>
`/usr/share/nginx/nextcloud/config/config.php`

```
'memcache.local' => '\OC\Memcache\APCu',
'memcache.locking' => '\OC\Memcache\Redis',
'memcache.distributed' => '\OC\Memcache\Redis',
'redis' => [
    'host' => '/var/run/redis/redis.sock',
    'port' => 0,
],
```

#### Tuning the Database
For normal databases smaller than 1GB set the following in the `[mysqld]` section of `/etc/my.cnf.d/mariadb-server.cnf`.

```
innodb_buffer_pool_size=1G
innodb_io_capacity=4000
```

Restart MariaDB and check if there is still sufficient free RAM, so that the system does not start to use the swap partition when it receives a burst of request.

#### Tuning PHP

Set the `memory_limit` in `/etc/php.ini` to at least 512M.

##### PHP-FPM
For example on a machine with 4G of RAM and 1G of MySQL cache configure following values in the `www.conf`.

```
pm = dynamic
pm.max_children = 120
pm.start_servers = 12
pm.min_spare_servers = 6
pm.max_spare_servers = 18
```

##### PHP OPCache
In `/etc/php.d/10-opcache.ini` set at least the following settings.

```
opcache.enable=1
opcache.interned_strings_buffer=8
opcache.max_accelerated_files=10000
opcache.memory_consumption=128
opcache.save_comments=1
opcache.revalidate_freq=1
```

Restart `php-pfm` and `nginx` services.

### Background Jobs
Nextcloud requires some tasks to be done on a regular basis. Cron is the most reliable method for doing that. 

Set up a cron job for `nginx` user

```
crontab -u nginx -e
```

and append this line

```
*/5 * * * * php -f /usr/share/nginx/nextcloud/cron.php
```

to run the tasks every 5 minutes.

Then make sure the method changes at the *Basic settings* admin page.

### Security
Since a Firewall and SELinux are active on the host running Nextcloud, only one further measure will be taken securitywise for now.

#### Deactivate nginx Default Site
To deactivate accessing any information about the system by making a request to the server IP, first deactivate the default site by removing the symlink to `default.conf` from the `sites-enabled` folder.<br/>
Then create a config file in `sites-available` and link it to `sites-enabled`. The files contents are:

```
server {
    listen 80;
    server_name "";
    return 444;
}
```


### Last Checks if Everything is Setup
Log into Nextcloud with the administrator account and navigate to **Settings -> Overview**. Follow the instructions and tips if there are any errors/warnings.

Go to **Settings -> Logging** and check for any errors.

Let Nextcloud scan your installation: https://scan.nextcloud.com/

A more sophisticated scan can be done by Mozilla: https://observatory.mozilla.org/analyze

### Automated Backup

Download or clone the repository with backup and restore scripts perfectly suited for this setup from<br/>
`https://codeberg.org/DecaTec/Nextcloud-Backup-Restore`

Open `NextcloudBackup.sh` in your text editor and customize all values which are marked with *TODO*.

After testing the script and with the Nextcloud instance still functional, create a cron job for the user root. Following line will run the backup script every Sunday at 02:00.

```
0 2 * * 0 /root/NextcloudBackupRestore/NextcloudBackup.sh
```


## Tips
To use the command occ: `sudo -u nginx php /usr/share/nginx/nextcloud/occ`. Add it to your aliases.

## Table of References
https://wiki.archlinux.org/index.php/Nextcloud and links

https://docs.nextcloud.com/server/18/admin_manual/contents.html and links

https://www.linuxbabe.com/redhat/install-lemp-nginx-mariadb-php7-rhel-8-centos-8

https://packages.debian.org/de/stretch/nginx-full

https://www.linuxbabe.com/redhat/install-nextcloud-rhel-8-centos-8-nginx-lemp

https://centosfaq.org/centos/yum-dnf-possible-confusion-centos-8/

https://certbot.eff.org/lets-encrypt/centosrhel7-nginx.html

https://stackoverflow.com/questions/26474222/mariadb-10-centos-7-moving-datadir-woes

https://decatec.de/home-server/nextcloud-auf-ubuntu-server-18-04-lts-mit-nginx-mariadb-php-lets-encrypt-redis-und-fail2ban
